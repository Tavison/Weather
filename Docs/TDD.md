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

The plugin imports no other plugin. It has zero dependencies on Soil,
Vegetation, Botany, or Phenology. Translation from `FWeatherSample` into soil
and plant input types is the responsibility of `UPhenologyWeatherLibrary` in
the Phenology plugin, which is the only code that imports Weather and
Soil/Vegetation types in the same translation unit.

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
clipped rather than wrapped. The downstream consumer converts it to vapour
pressure deficit: `VPD_kPa = SVP(T) × (1 − RH)`.

AR(1) was chosen over i.i.d. because independent days produce unrealistically
rapid swings — a −15°C day followed by a +10°C day the next. The model's
simplicity is deliberate: a two-parameter per-channel setup (mean curve + α
scalar) that designers can tune without a statistics background.

### Rainfall — Bernoulli × Normal, i.i.d.

Each day is an independent two-stage draw:
1. Bernoulli trial against `RainfallProbability(day)` — is it a wet day?
2. On wet days: amount drawn from `Normal(RainfallMean_mm, RainfallStdDev_mm)`,
   clamped to ≥ 0.

Rainfall is deliberately not AR(1) in V1. The reasoning: daily rainfall
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

## Determinism — per-day hashing and local random streams

`AdvanceTo(T_end)` is deterministic from
`(Template, Seed, LatitudeDeg, LastProcessedTime, T_end)`. Per-day RNG is
keyed by a pure hash:

```cpp
static int32 MakeDaySeed(int32 GlobalSeed, int32 DayIndex, int32 Channel)
{
    const uint32 A = static_cast<uint32>(GlobalSeed);
    const uint32 B = static_cast<uint32>(DayIndex * 10 + Channel);
    return static_cast<int32>(A ^ (B * 2654435761u));
}
```

`MakeDaySeed` is a pure function — no state, no side effects. For each day
and each channel, a local `FRandomStream` is created from the day seed, one
value is drawn, and the stream is immediately destroyed:

```cpp
FRandomStream R0(MakeDaySeed(Seed, DayIndex, Slot0));  // local variable
float TempNoise = R0.GetFraction();                      // one draw
// R0 is destroyed here — nothing persists
```

No `FRandomStream` is ever stored as a member. The AR(1) carry (last
temperature, last wind speed, last relative humidity) is stored as plain
floats, not as a random stream — the carry reflects where the Markov chain
is, not an ongoing random state. Any sub-span produces the same values it
would produce inside any other call; span boundaries are not special.

After each successful `AdvanceTo`, the final AR(1) carry is stored. The next
call forwards that carry directly — all three channels are bit-exactly
continuous at every span boundary. Calls that advance week by week produce
exactly the same output as a single call covering the full period.

`Initialize` and `SetLastProcessedTime` both clear the carry. The next call
falls back to a 30-day burn-in: the AR(1) state is bootstrapped by running
30 days of draws before `LastProcessedTime` using `Hash(Seed, day_index)`
for those pre-history days. The burn-in produces a carry that is statistically
representative of the climate at the repositioned time, but is not identical
to what a continuously-advanced sequence would have produced. This seam is
small and typically unobservable over the seasonal signal.

**Seed ownership.** The seed is caller-supplied — the plugin guarantees
determinism given a seed, but does not generate seeds. Using a stable entity
hash (e.g. `GetTypeHash(RegionGUID) ^ WorldSeed`) means the same region
always produces the same climate sequence across saves. Using `FMath::Rand()`
produces non-deterministic sequences. Using a fixed seed (0) makes every
region identical.

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

## Plugin boundary — zero cross-plugin dependencies

The Weather plugin — `UWeatherTemplate`, `UWeatherInstance`, `FWeatherSample`
— has no knowledge of Soil, Vegetation, Botany, or Phenology. It imports no
other plugin module. Projects that use Weather for non-agriculture purposes
(e.g. weather-driven terrain or visual effects) can depend on only the
`Weather` module and omit everything else.

Translation from `FWeatherSample` into soil and plant input types is entirely
owned by `UPhenologyWeatherLibrary` in the Phenology plugin. That class is the
only place where Weather output types and Soil/Vegetation input types appear in
the same file. The four translation functions it owns are:

- `WeatherSampleToSoilWeather` — fills `FSoilWeather` (curve-path soil input)
- `WeatherSampleToSoilWeatherInput` — fills `FSoilWeatherInput` (tick-path soil input)
- `WeatherSampleToVegetationAmbient` — fills five channels of `FVegetationAmbient` (AirTemperature, DayLength, Wind, VPD_kPa, PAR)
- `WeatherSampleToVegetationTickAmbient` — fills `FVegetationTickAmbient` (tick-path plant input)

## Known limitations

**Rainfall is i.i.d.** Multi-day rainy stretches and dry spells do not have
day-to-day correlation in V1. Extended spell modelling via regime Markov
chains is the Tier 3 extension.

**No heatwave or cold-snap overlay.** AR(1) smoothing produces persistence
but does not model the distinct meteorological phenomenon of a stagnant
high-pressure system holding temperatures well above or below normal for a
week.

**No cross-year climate drift.** Each year draws from the same template
curves. Multi-decadal trends, El Niño cycles, or scenario-based climate
change are not modelled.

**No terrain microclimate.** A single `UWeatherInstance` produces one climate
at one latitude. Valley frost pockets, upland exposure, coastal moderation,
and rain-shadow effects require multiple instances with different templates
and/or post-processing on the generated values.

## Test surface

Fourteen automation tests across two files. Seven translation tests that were
previously in `WeatherBlueprintLibraryTests.cpp` have moved to
`Phenology.PhenologyWeatherLibrary.*` in the Phenology plugin — they test the
Phenology boundary function, not Weather internals.

| Suite | File | Count | Coverage |
|---|---|---|---|
| `Weather.WeatherTemplate.*` | `WeatherTemplateTests.cpp` | 3 | Template construction, field validation, AR(1) scalars |
| `Weather.WeatherInstance.*` | `WeatherInstanceTests.cpp` | 11 | AR(1) continuity, burn-in seam, per-day determinism, cursor semantics, RH range and continuity |

### WeatherTemplate tests (3)

| Test | What it verifies |
|---|---|
| `CDO.HasExpectedDefaults` | Class defaults are non-null and sane (α in [0,1], σ > 0) |
| `Curves.AcceptKeysAndRoundTrip` | Keys written to template curves survive round-trip via `GetRichCurve()` |
| `AR1.ScalarsStoreFreely` | AR(1) alpha fields accept any float and read back the written value |

### WeatherInstance tests (11)

| Test | What it verifies |
|---|---|
| `CDO.HasNullTemplate` | Uninitialized instance has null template and zero seed |
| `Initialize.BindsTemplateAndScalars` | `Initialize` binds template, seed, and latitude; cursor at 0 |
| `AdvanceTo.PastOrEqualTEndIsNoOp` | `NowTime ≤ LastProcessedTime` → no output keys emitted, cursor unchanged |
| `AdvanceTo.ChannelKeyCountMatchesSpanDays` | 10-day advance → 10 keys per output channel |
| `AdvanceTo.Deterministic` | Same seed + same span → bit-identical output on two calls |
| `AdvanceTo.DayLengthAtEquatorIsNoon` | Latitude 0 at equinox → day length ≈ 12 hours |
| `AdvanceTo.SequentialCallsAreBitIdentical` | Two sequential advances over [0,5] and [5,10] match one advance over [0,10] |
| `SetLastProcessedTime.ResetsBurnIn` | Repositioned cursor triggers burn-in; next advance differs from a continuously-advanced sequence but not identically |
| `AdvanceTo.RepositionedSpanApproxContinuity` | Repositioned then advanced span is within ±3σ of the continuously-advanced sequence |
| `AdvanceTo.RelativeHumidityClampedToUnitInterval` | High-σ RH draw is always in [0, 1] |
| `AdvanceTo.RelativeHumidityContinuousAcrossSpans` | RH is bit-exactly continuous at sequential span boundaries |
