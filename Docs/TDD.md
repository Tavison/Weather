# Weather — Technical Design

Stochastic daily weather generation for Unreal Engine 5.

## Purpose

The Weather plugin separates climate description from weather realisation.
A `UWeatherTemplate` encodes the statistical envelope of a locale: what
temperatures, rainfall amounts, and wind speeds to expect on any given
day-of-year. A `UWeatherInstance` draws a specific realisation from those
statistics — a particular year's sequence of daily values — governed by a
seed and a latitude. The output is `FWeatherSample`, an `FRuntimeFloatCurve`
per channel over the requested span, ready for downstream consumers.

The plugin imports no other agriculture plugin. Soil and Vegetation receive
an `FWeatherSample` and field-copy from it via `UWeatherBlueprintLibrary`;
neither knows anything about the generation process.

## Stochastic models

### Temperature, wind, and relative humidity — AR(1)

Temperature, wind, and relative humidity use a first-order autoregressive
(AR(1)) model:

```
Today = α · Yesterday + (1 − α) · Mean(day_of_year) + Normal(0, σ(day_of_year))
```

`α` is the autocorrelation coefficient — `TemperatureAR1Alpha`, `WindAR1Alpha`,
and `RelativeHumidityAR1Alpha` on the template. At `α = 0` days are fully
independent; at `α = 1` the channel is a random walk that never reverts to the
seasonal mean. Realistic temperate values are 0.5–0.7 for temperature (cold
fronts linger 2–4 days), 0.3–0.5 for wind (storms pass faster), and 0.5–0.7
for humidity (dry or humid air masses persist for several days).

Relative humidity is emitted as a fraction in `[0, 1]` and clamped to that
range after each AR(1) draw — draws that stray outside due to large σ are
clipped rather than wrapped. The downstream consumer (Botany boundary) converts
it to vapour pressure deficit: `VPD_kPa = SVP(T) × (1 − RH)`.

AR(1) was chosen over i.i.d. because independent days produce unrealistically
rapid swings — a −15°C day followed by a +10°C day the next. The model's
simplicity is deliberate: a two-parameter per-channel setup (mean curve + α
scalar) that designers can tune without a statistics background.

### Rainfall — Bernoulli × Normal, i.i.d.

Each day is an independent two-stage draw:
1. Bernoulli trial against `RainfallProbability(day)` — is it a wet day?
2. On wet days: amount drawn from `Normal(RainfallMean_mm, RainfallStdDev_mm)`,
   clamped to ≥ 0.

Rainfall is deliberately NOT AR(1) in V1. The reasoning: daily rainfall
independence is a reasonable first approximation for seasonal simulations,
and the added complexity of multi-day regime modelling (Markov chains for
storm systems, monsoon weeks) is deferred to Tier 3. It is a known limitation
that consecutive days in extended rainy spells or dry spells are not
correlated.

### Day length and solar insolation — closed-form

Day length is computed from latitude and day-of-year using the standard
sunrise hour-angle formula (`acos(-tan(lat) · tan(declination))`). Solar
insolation is estimated as `DayLength_hours × 2.5 MJ/m²/hr`, attenuated
by a cloud-cover factor: `(1 - RainfallProbability) + 0.3 × RainfallProbability`.
An overcast sky transmits approximately 30% of clear-sky radiation.

Neither channel appears on `UWeatherTemplate` — designers are not asked to
author astronomy. The runtime computes both from the latitude passed to
`Initialize`, so a single template can be used at any latitude and produce
correct seasonal day lengths for that site.

## Determinism and carry state

`AdvanceTo(T_end)` is deterministic from
`(Template, Seed, LatitudeDeg, LastProcessedTime, T_end)`. Per-day RNG is
keyed by `Hash(Seed, day_index)`, so any sub-span produces the same values
it would produce inside any other call — the span boundaries are not
special.

After each successful `AdvanceTo`, the final AR(1) state (temperature, wind,
and relative humidity carry) is stored. The next `AdvanceTo` call forwards
that carry directly — all three channels are bit-exactly continuous at every
span boundary.
Calls that advance week by week produce exactly the same output as a single
call covering the full period.

`Initialize` and `SetLastProcessedTime` both clear the carry. The next call
falls back to a 30-day burn-in: the AR(1) state is bootstrapped by running
30 days of draws before `LastProcessedTime` using `Hash(Seed, day_index)`
for those pre-history days. The burn-in produces a carry that is statistically
representative of the climate at the repositioned time, but is not identical
to what a continuously-advanced sequence would have produced. This seam is
small and typically unobservable over the seasonal signal.

## Template curve convention

All template curves are keyed on day-of-year `[0, 365]`. The runtime
evaluates each curve at `day_of_year = DayIndex % 365` where `DayIndex`
is the integer game-time day. `FRuntimeFloatCurve` extrapolates by clamping
at the boundary key, so a curve that only covers `[0, 365]` handles
multi-year simulations by repeating the final key — which is correct
for perennial annual-cycle climates.

σ (standard deviation) curves should never go negative. The runtime takes the
absolute value defensively, but a negative σ indicates a confused authoring
intent. AR(1) coefficients in `[0, 1]` are clamped defensively; values
outside that range produce non-stationary sequences but will not crash.

## Plugin boundary

`UWeatherBlueprintLibrary` is the only file in the Weather plugin that imports
Soil or Vegetation types. It is a translation layer: field-copy from
`FWeatherSample` channels into `FSoilWeather`, `FSoilWeatherInput`,
`FVegetationAmbient`, and `FVegetationTickAmbient`. The Weather plugin itself
— `UWeatherTemplate`, `UWeatherInstance`, `FWeatherSample` — has no knowledge
of what consumes its output.

This separation means the Weather plugin ships and compiles without Soil or
Vegetation. Projects that use Weather for non-agriculture purposes can depend
on only the `Weather` module and omit `UWeatherBlueprintLibrary`.

## Known limitations / deferred work

**Rainfall is i.i.d.** Multi-day rainy stretches and dry spells do not have
day-to-day correlation in V1. Extended spell modelling via regime Markov
chains is the Tier 3 extension.

**No heatwave or cold-snap overlay.** AR(1) smoothing produces persistence
but does not model the distinct meteorological phenomenon of a stagnant high-
pressure system holding temperatures well above or below normal for a week.
A heatwave / cold-snap regime layer is a potential future extension.

**No cross-year climate drift.** Each year draws from the same template curves.
Multi-decadal trends, El Niño cycles, or scenario-based climate change are not
modelled. A template-blending or curve-drift mechanism is a potential future
extension for projects that need long-horizon agricultural simulation.

**No terrain microclimate.** A single `UWeatherInstance` produces one climate
at one latitude. Valley frost pockets, upland exposure, coastal moderation, and
rain-shadow effects require multiple instances with different templates and/or
post-processing on the generated values.

## Test surface

Twenty-one automation tests across three files in
`Plugins/Weather/Source/Weather/Private/Tests/`.

| Suite | File | Covers |
|---|---|---|
| `Weather.WeatherTemplate.*` | `WeatherTemplateTests.cpp` | Template construction, field validation, RH curves |
| `Weather.WeatherInstance.*` | `WeatherInstanceTests.cpp` | AR(1) continuity, burn-in seam, determinism, cursor semantics, RH range + continuity |
| `Weather.WeatherBlueprintLibrary.*` | `WeatherBlueprintLibraryTests.cpp` | Field-copy correctness, unfilled channel contracts, VPD formula + boundary conditions |
