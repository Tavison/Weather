# Weather: Technical Design

This document explains how `UWeatherInstance` turns a `UWeatherTemplate` into
weather, and why it is built the way it is. Read the [main page](../mainpage.md)
first for how to call it; this page is for someone changing the generator,
debugging odd output, or deciding whether a behavior is a bug or a decision.

## The shape of the design

Weather is a producer that holds one day. `Initialize` binds a template and
generates the starting day. `AdvanceDay` throws that day away and generates the
next. There is no cursor, no span, and no fractional-day advance. A consumer that
wants sub-day granularity samples the hourly curves of the day it is holding.

That shape drives everything else:

- **State is carried, not replayed.** The random deviations that make Tuesday
  resemble Monday, and the front that is still raining at midnight, live in a
  handful of private members that survive from one `AdvanceDay` to the next.
  Nothing can jump to an arbitrary day, because nothing stores the days it
  skipped.
- **One random stream serves every channel.** See [Random numbers](#random-numbers-and-determinism).
- **Compression is not this plugin's business.** Weather generates exactly one real
  solar day per call and has no notion of pacing against real time. A consumer
  that wants a compressed calendar decides how many days to pull per unit of game
  time. Do not add a ratio parameter here; one was removed on purpose.

The result type, `FWeatherDay`, and the `IWeatherDaySource` interface live in
`EcologyCommon` rather than in this plugin, so Phenology and `WeatherGameplay` can
depend on the data shape without depending on this implementation. `UWeatherInstance`
implements `IWeatherDaySource`.

## What happens in one day

`GenerateDay` (in `WeatherInstance.cpp`) builds one `FWeatherDay` from the template
and the carried state. Day-of-year is `(Day − 1) mod YearLength`, zero-based, and
it indexes every template curve. Then, for each of 24 hours, in this fixed order:

| Step | Draw from `Rng` | Produces |
|---|---|---|
| 1. Temperature | 1 Gaussian (2 fractions) | Steps the temperature deviation; adds mean and diurnal wave. |
| 2. Solar elevation | none | True geometric elevation at this hour. |
| 3. Humidity | 1 Gaussian (2 fractions) | Steps the humidity deviation; clamps to `[0, 1]`. |
| 4. Microclimate | 1 Gaussian (2 fractions) | Steps the microclimate deviation; blends typical days. |
| 5. Front | 1 fraction while no front is active, plus up to 3 more on the hour one begins | Advances the front; yields its intensity. |
| 6. Wind | 1 Gaussian (2 fractions) | Steps the wind deviation; adds any front boost. |
| 7. Precipitation | none | Sums microclimate and front; splits rain from snow by temperature. |
| 8. Snow depth, blizzard | none | Derived from the split. |
| 9. Insolation | none | Clear-sky irradiance from step 2, attenuated by steps 3 and 7. |

Steps 2 and 7 through 9 draw nothing. Insolation has to come last because it reads
that hour's generated humidity and precipitation. `DayLength_hours` is a scalar
computed once, before the loop.

`Initialize` runs the same pipeline once. Before it does, it constructs `Rng`
fresh from the seed and bootstraps each deviation process by drawing from its
stationary distribution, in the order temperature, humidity, microclimate, wind.
Front state starts inactive. It then generates the starting day.

## Random numbers and determinism

Every stochastic draw comes from one private `FRandomStream`, consumed
sequentially in the order above. Gaussians come from Box-Muller, which consumes two
fractions each.

An earlier version hashed a per-channel, per-hour seed so any hour could be
regenerated independently. That bought random access to any day, and nothing in
this design ever needed it, because `Initialize` always starts a fresh stream.
The hash scheme's slot bookkeeping was pure cost, so it was deleted. A shared
sequential stream has no slot collisions to manage, and the variable draw count
in the front logic (only some hours draw a duration) needs no reserved slots.

Consequences worth knowing:

- **Same nonzero seed, same template, same `FWeatherInitParams` gives identical
  output**, day after day. The tests compare two instances in one process.
  Cross-platform or cross-compiler bit equality is neither claimed nor tested.
- **A saved seed is only valid for the code that produced it.** The stream is
  consumed in a fixed order, so adding, removing, or reordering any draw changes
  every number downstream of it. Treat a seed as a reproduction key for a given
  build of the generator, not a permanent identifier.
- **`StartingDay` is not a seek.** Initializing at day 200 starts a fresh stream at
  day 200. It is a plausible day 200, not the one you would have reached from day 1.
- **Seed 0 means randomize.** `Initialize` draws `GetTypeHash(FGuid::NewGuid())`,
  and substitutes 1 if that happens to be 0, so a stored seed is never the
  "randomize" sentinel. The source is deliberately not `FMath::Rand()` or any
  shared RNG: consuming a draw from state that some other system may be managing
  for its own reproducibility would be a silent side effect. The seed actually used
  is stored and read back with `GetSeed()`.

## Temperature, humidity, and wind: a continuous mean-reverting process

Each of these channels is its seasonal mean plus a deviation. The deviation is an
Ornstein-Uhlenbeck process, `dX = −θX dt + σ dW`, stepped once per hour using the
exact discrete transition rather than an Euler approximation:

```
X' = X · e^(−θΔt) + sqrt( σ² (1 − e^(−2θΔt)) / (2θ) ) · Z        Z ~ N(0, 1), Δt = 1/24 day
```

`θ` is 0.2 per day, so a deviation decays with an e-folding time of five days. That
is what makes a cold snap last a few days and then relax, without any explicit
"cold snap" object. The exact transition matters because the step is evaluable at
any `Δt`, and it makes the process's long-run behavior independent of the step size.

The stationary standard deviation of the process is `σ / sqrt(2θ)`, about `1.58σ`
at `θ = 0.2`. **The value a designer authors is `σ`, the noise scale, not the
spread they will observe.** The observed spread is roughly 1.6 times larger. The
same applies to humidity, wind, and microclimate. This is inherent in the model
rather than a bug, but the template's field comments call these "standard
deviations" and a designer will reasonably expect that to be literal.

Carried state per channel is one float. It is set by the stationary draw at
`Initialize` and by the previous day's last hour afterward, so the deviation is
continuous across midnight. The seasonal mean and the diurnal wave are re-evaluated
each day, so the *value* may step at midnight even though the *deviation* does not.

**Temperature** is `mean(day-of-year) + deviation + wave · amplitude`. The wave is
two half-cosines, one rising from the 06:00 trough to the 15:00 peak and one
falling back over the remaining fifteen hours, continuous at both anchors. The
amplitude is `6 °C × (1 − 0.5) = 3 °C`, where 0.5 is a flat placeholder standing
in for generated humidity. The noise scale is a fixed 1.5 °C. The template's
`TemperatureStdDev_C` is not consumed.

**Relative humidity** is `mean + deviation`, clamped to `[0, 1]`, with `σ` read
from `RelativeHumidityStdDev` at that day's day-of-year. It has no diurnal wave.

**Wind** is `mean + deviation + boost`, clamped at zero. The boost is the active
typical front's `WindBoost_ms`, added flat for as long as that front is active.
Microclimate never adds wind, because a persistent local drizzle is not a wind
event.

## Rainfall: two mechanisms, summed

Rainfall is not one archetype pick. A persistent local pattern and a passing system
are different physical things that can happen at once, so they are generated
separately and added. Neither suppresses the other.

**Microclimate** is the persistent baseline. Each hour, the target intensity is a
weighted blend of the template's `MicroclimateTypicalDays`, each weighted by its own
`SeasonalLikelihood` at the day-of-year and normalized here (weights need not sum to
1). No typical days, or all weights zero this time of year, blends to zero, so an
unauthored microclimate contributes nothing. A microclimate deviation from the same
kind of mean-reverting process rides on top, and the total is clamped at zero. Dry
hours and multi-day dry or wet stretches therefore emerge from the process dipping
the blend to zero and reverting slowly. There is no per-day "does it rain" coin
flip, and the spread is deliberately not derived from disagreement between typical
days, which would leave a season dominated by one shape with no variability.

**Fronts** are events with a start, a duration, and a shape. While none is active,
each hour rolls a hazard check. `FrontFrequency` is authored as a daily likelihood
`p` and converted to an hourly hazard `1 − (1 − p)^(1/24)`, evaluated continuously
rather than once per calendar day, so a front spilling past midnight never
suppresses that day's own independent chance at another. When one begins, a typical
front is drawn by `SeasonalLikelihood` and a duration is drawn from
`FrontMeanDurationHours` and `FrontDurationStdDevHours`, floored at 0.5 hours.
Shape and duration are drawn independently, which is the point: the same authored
character stretches into a short pop-up cell or a multi-day soaking. Intensity is
the typical front's `IntensityByPassageFraction` evaluated at
`elapsed / duration`. At most one front is active at a time, and its state
(`FWeatherFrontRuntimeState`) is carried across the day boundary with no
truncation.

**Snow is a gate, not a mechanism.** The hour's total precipitation goes wholly to
`Snowfall_mm` if that hour's temperature is below `SnowTemperatureThresholdC`, and
wholly to `Rainfall_mm` otherwise, so the two curves are mutually exclusive at
every key and sum to the total.

**`SnowDepthRate_cm`** converts water-equivalent snowfall to depth using a
temperature-dependent snow-to-liquid ratio in
`UWeatherDayLibrary::SnowWaterEquivalentToDepthCm` (EcologyCommon), so consumers
never need to know the conversion exists. It is a rate, not a running total.
There is deliberately no ground snowpack or melt model here; that is a separate,
undesigned feature.

**`IsBlizzard`** is true when it is snowing, a front (not microclimate alone) is
driving it, and the front's intensity this hour is at least
`BlizzardIntensityFraction` (0.7) of that typical front's own authored peak,
found with `FRichCurve::GetValueRange`. The threshold is relative on purpose, so a
violent front and a gentle one are each judged against themselves rather than
against an absolute mm-per-hour number.

Precipitation, snow, snow depth, blizzard, and insolation curves use linear keys.
Temperature, humidity, wind, and solar elevation use smooth (cubic, auto-tangent)
keys. Rain would overshoot below zero with cubic interpolation at a sharp edge, and
a negative intensity is meaningless.

## Solar geometry and insolation

Neither is stochastic, and neither draws from `Rng`.

**Declination** is `AxialTiltDeg · sin(2π (284 + d) / YearLength)` for 1-based
day-of-year `d`, after Cooper (1969). It divides by the runtime's `YearLength`, not
365, so the solar cycle stays phase-locked to the template's annual curves under a
compressed calendar. `AxialTiltDeg` lives on `FWeatherInitParams`, not the template,
because it belongs to the planet and not to a locale's climate.

**Elevation** at an hour is `asin(sin φ sin δ + cos φ cos δ cos h)` with hour angle
`h = 15° · (hour − 12)`. It is true geometry, with no obstruction and no refraction,
and it is negative at night.

**Day length** is `2 · acos(cosH) / 15` hours, where
`cosH = (sin(θ_h) − sin φ sin δ) / (cos φ cos δ)` and the threshold
`θ_h = −0.833° + HorizonObstructionDeg`. The 0.833° is standard almanac refraction
plus the sun's angular radius, so an unobstructed site matches real sunrise and
sunset. `HorizonObstructionDeg` is added on top of that, not used in its place, and
applies symmetrically to sunrise and sunset. A `cosH` above 1 means the sun never
clears the horizon (0 hours) and below −1 means it never sets (24 hours).
`Initialize` clamps latitude to `[−89.9°, 89.9°]` so `cos φ` stays positive.

**Insolation** is clear-sky irradiance,
`1361 W/m² · max(sin elevation, 0) · 0.75`, multiplied by an attenuation factor
computed from the same hour's generated weather. If anything is precipitating, the
factor is a flat 0.3. Otherwise humidity above 0.6 lerps the factor from 1 down to
0.85 at full saturation. Precipitation wins over haze; the two are not combined.
Output is an instantaneous rate in W/m², not a daily total, so no day-length
division is needed to derive PAR from it.

## Errors and logging

Weather uses the shared error-handling policy from `STD_Utilities`, wrapped as
`Weather::EH`. Nothing writes a naked `throw`; everything goes through `EH::Throw`.
With exceptions enabled it throws. With them disabled, the Unreal default, it logs
to `LogWeather`, and the two call sites that cannot continue (a null template in
`Initialize`, and `AdvanceDay` before `Initialize`) follow that with a Fatal log.

| Exception | Raised when |
|---|---|
| `Weather::NullTemplateException` | `Initialize` is passed a null template. |
| `Weather::NotInitializedException` | `AdvanceDay` runs before `Initialize`. |

`GetCurrentDay`, `GetTodaysWeather`, and the other getters do not check. Before
`Initialize` they return day 1 and an empty `FWeatherDay`.

## Plugin boundary

Among project plugins, Weather depends on `EcologyCommon` and `STD_Utilities`, and
on the engine's `GameplayTags` module. It has no knowledge of Soil, Vegetation,
Botany, or Phenology, and there is no module-level state. `UPhenologyWeatherLibrary`
(Phenology) translates `FWeatherDay` into Soil and Vegetation inputs, so that is the
only place Weather output and those consumer types appear together.

## Known limitations and open design debt

These are known and mostly deliberate. Several have their reasoning recorded next
to the code; do not "fix" them without reading it.

- **`TemperatureStdDev_C` is authored but not read.** Temperature's noise scale is a
  fixed 1.5 °C (`DeviationSigma_C`). Deriving it from the curve is a recorded TODO.
- **Temperature's diurnal wave uses fixed anchors and a placeholder humidity.**
  Trough 06:00, peak 15:00, amplitude scaled by a flat 0.5 rather than the
  generated humidity. Deriving the anchors from the solar geometry is not a direct
  substitution, since temperature peaks after solar noon, and re-timing it would
  ripple into the snow/rain gate and blizzard classification, which key off
  temperature.
- **The process constants are untuned.** `θ = 0.2/day`, the 1.5 °C scale, the 6 °C
  base amplitude, the 0.7 blizzard fraction, and the cloud and haze constants are
  chosen for plausibility, not fitted to data.
- **`LightningProbability` is authored and unused.** Nothing consumes it yet.
- **No gusts.** Wind is sustained speed plus a front's flat boost. Gusts are meant
  to be a front-like event mechanic authored by frequency and strength ranges, not a
  wider standard deviation, and need their own design pass.
- **Insolation simplifications.** It ignores orbital eccentricity (real
  extraterrestrial irradiance varies about ±3.3% over the year) and uses a flat
  clear-sky transmittance, so dawn and dusk read brighter than a real air-mass model
  would give.
- **Day length at 90° tilt.** With `AxialTiltDeg` at its clamp maximum of 90°, a
  solstice can put declination at exactly ±90°, where the day-length denominator
  `cos φ cos δ` reaches zero. No test covers tilts that extreme.
- **No cross-year drift, no heatwave overlay, no terrain microclimate.** Every year
  draws from the same curves. Persistence comes only from mean reversion. One
  instance is one climate at one site, apart from `HorizonObstructionDeg`.
- **No ground snowpack or melt.**

## Test surface

Forty-six tests across two files, all under `Weather.*`. Run them headless with
`Automation RunTests Weather.WeatherInstance` and `Weather.WeatherTemplate`.

| Suite | Count | Covers |
|---|---|---|
| `WeatherTemplate.*` | 2 | Class-default values, and key round-trip through the curve wrappers. |
| `WeatherInstance.*` | 44 | Everything else, grouped below. |

| `WeatherInstance` group | Count | What it establishes |
|---|---|---|
| Lifecycle and determinism | 7 | Null template before `Initialize`; explicit and default values bind; today exists at once; `AdvanceDay` steps the day by one; identical inputs on two instances give identical days; an omitted or zero seed randomizes per instance while an explicit one is stored verbatim. |
| Temperature and humidity | 3 | Temperature varies within a day; humidity stays in `[0, 1]`; humidity equals its mean curve when the deviation is zero. |
| Microclimate | 4 | Zero rainfall with no typical days; single-day shape reproduced; weighted blend; never negative. |
| Front | 4 | Fires and stays bounded by its authored intensity; follows the authored shape across a passage; spans midnight without truncation; one shape stretches to different durations. |
| Snow and blizzard | 7 | The rain/snow split at impossible thresholds and against real temperature; no blizzard from microclimate alone or without snow; blizzard tracks the front's own peak; depth rate matches its formula. |
| Wind | 3 | Follows the mean curve with zero deviation; never negative; boosted while a front is active. |
| Solar geometry | 7 | Equator and equinox day length; solstice asymmetry at mid-latitude; noon peak elevation matches the closed form; obstruction of zero equals the default and a positive value strictly shortens the day; polar latitudes never yield NaN. |
| Insolation | 6 | Zero at night; matches the clear-sky formula when dry and clear; darker while precipitating; never above clear-sky or below zero; summer noon exceeds winter noon; non-negative at half-hour samples. |
| Axial tilt | 3 | Omitted tilt matches Earth's default; zero tilt removes the seasonal day-length swing; higher tilt makes solstices more extreme. |

Tests that need exact values set the relevant deviation to zero so the outcome
does not depend on a seed. The solar and insolation tests are seed-independent by
construction, and their consuming no draws from `Rng` is what let the earlier tests
stay unchanged when those channels were added.
