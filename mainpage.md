\mainpage Weather Plugin

Generates believable weather, one day at a time, from a climate you author.

## Overview

Reach for this plugin when something in your game needs weather that behaves
like weather: cold snaps and warm spells that last a few days, drizzle that
comes and goes, fronts that roll through and take the wind with them, and
seasons whose length depends on where you are on the planet. You describe a
place once, as a `UWeatherTemplate` data asset. A `UWeatherInstance` binds that
template, and every call to `AdvanceDay` hands you a new day of hourly curves:
temperature, humidity, wind, rain, snow, blizzard flag, sun elevation, and solar
irradiance, plus the day's length in hours.

The plugin has no cursor and no clock. It always holds exactly one day, the
"current" day, and a new day exists only when you ask for it. Pace and time
compression are your decisions, not the plugin's.

Among the project's plugins, Weather depends only on `EcologyCommon` (for the
`FWeatherDay` result type and the `IWeatherDaySource` interface) and
`STD_Utilities`. It knows nothing about Soil, Vegetation, or Botany, so it is
equally useful for driving visual effects or ambience.

## The primary path

```cpp
// 1. Author a UWeatherTemplate data asset in the editor (one per locale).
//    Create an instance and bind it. Only the template is mandatory; every
//    field of FWeatherInitParams has a default, so override what you need.
UWeatherInstance* Weather = NewObject<UWeatherInstance>(Outer);
Weather->Initialize(WeatherTemplate, { .Seed = 543243524, .Latitude = 51.5f, .StartingDay = 93 });

// 2. Today's weather exists the moment Initialize returns. No AdvanceDay needed.
const FWeatherDay& Today = Weather->GetTodaysWeather();

// 3. Sample any hourly channel at any time of day, not just on whole hours.
const float NoonTempC   = Today.Temperature_C.GetRichCurveConst()->Eval(12.0f);
const float NoonWind_ms = Today.WindSpeed_ms.GetRichCurveConst()->Eval(12.0f);
const float DaylightHrs = Today.DayLength_hours;             // a scalar, not a curve

// 4. Move to the next day. Same value GetTodaysWeather() now returns.
const FWeatherDay Tomorrow = Weather->AdvanceDay();
// Weather->GetCurrentDay() is now 94.
```

Every hourly channel is a `FRuntimeFloatCurve` keyed on hour-of-day `[0, 24)`, so
you sample it with `GetRichCurveConst()->Eval(Hour)` at whatever rate your system
runs. Keys sit on whole hours 0 through 23 and the curve holds its last value
until hour 24, so a sample at 23.5 returns the hour-23 value.

**`StartingDay` is 1-based: day 1 is 1 January.** Days past `YearLength` wrap, so
day 366 reads the same climate curves as day 1.

### Handing weather to consumers

Depend on `IWeatherDaySource` (EcologyCommon), not on `UWeatherInstance`. It
exposes exactly the three calls above (`GetCurrentDay`, `GetTodaysWeather`,
`AdvanceDay`), and `UWeatherInstance` implements it. Phenology takes one as an
injected dependency and drives it for you:

```cpp
UPhenologyField* Scene = NewObject<UPhenologyField>(Outer);
Scene->Initialize(Weather);   // converts to TScriptInterface<IWeatherDaySource>
```

Translating a `FWeatherDay` into Soil and Vegetation input structs is done by
`UPhenologyWeatherLibrary` in the Phenology plugin. Weather itself never sees
those types.

For gameplay code, the `WeatherGameplay` plugin wraps a `UWeatherInstance` in a
world subsystem so you don't construct one by hand.

## Seeds and reproducing a run

Leave `Seed` at 0 and each `Initialize` draws its own random seed, so a new
session gets new weather. To repeat a run exactly, pass the same nonzero seed
with the same template and the same `FWeatherInitParams`. To capture the seed of
a run you didn't choose, call `GetSeed()` right after `Initialize`. It returns
the seed actually in use, never 0, and that is the value to save.

Determinism holds for a given build. The tests compare two instances built
from the same inputs in one process; nothing checks output across platforms or
compiler versions.

## Configuring a locale

A `UWeatherTemplate` holds only what varies by place. Everything astronomical
comes from `FWeatherInitParams` instead, because it belongs to the planet and
the site, not the climate.

**Temperature, humidity, wind.** Each has a mean curve, which sets the seasonal
shape, and a standard-deviation curve, which scales the random wander around that
mean. A wide winter value and a narrow summer one give you the changeable winters
and settled summers of a temperate climate. Temperature is the exception for now:
its wander uses a fixed 1.5 °C scale, and `TemperatureStdDev_C` is authored but not
yet read.

**Rain and snow are two independent mechanisms.** *Microclimate* is a
persistent local pattern, like an evening drizzle, and you author it as one or
more example days (`FWeatherTypicalDay`), each drawn as rainfall by hour. A
*Front* is a passing system, and you author it as one or more example passages
(`FWeatherTypicalFront`), each drawn as rainfall against fraction of the
passage. Both can be active in the same hour and neither suppresses the other;
the hour's rainfall is their sum. Snow is not a third mechanism. It is a
temperature gate: if the hour's temperature is below `SnowTemperatureThresholdC`,
the whole total lands in `Snowfall_mm` instead of `Rainfall_mm`.

**Day length and solar irradiance are not authored.** Both come from geometry:
`Latitude`, the day of year, `AxialTiltDeg`, and `HorizonObstructionDeg` (see the
reference below). The same template therefore gives correct seasons at any
latitude.

## Gotchas

**`StartingDay` is not a seek.** `Initialize` always starts a fresh random
stream, so initializing at day 200 gives you a plausible day 200, not the day 200
you would have reached by advancing from day 1 with the same seed. If you need
to resume a run, either advance from the original start or persist the state
you care about yourself.

**Only `AdvanceDay` checks for a missing `Initialize`.** Calling it first raises
`Weather::NotInitializedException`. `GetCurrentDay` and `GetTodaysWeather` do not
check: before `Initialize` they return day 1 and an empty `FWeatherDay`. A null
template raises `Weather::NullTemplateException`. With exceptions enabled these
throw; in a default Unreal build (exceptions off) they log and then hit a Fatal
log, so treat both as programming errors.

**`YearLength` is not validated.** Pass a value of 1 or more. The plugin does no
range check on `YearLength` or `StartingDay`; the other `Initialize` fields are
clamped (see the reference).

**Precipitation curves are rates in water-equivalent mm per hour, not daily
totals.** Integrate the curve over the day for a total. `Snowfall_mm` is water
equivalent, not depth. Use `SnowDepthRate_cm`, which is also a rate (depth added
during that hour), never an accumulating snowpack. Nothing here models snow on the
ground or melt.

**`IsBlizzard` is a curve.** It is `1.0` or `0.0` at each key. Treat any sampled
value above 0.5 as true.

**Sun elevation ignores your horizon; day length does not.** `HorizonObstructionDeg`
shortens `DayLength_hours` only. `SolarElevationDeg` is always the true geometric
angle.

**The standard deviations you author are noise scales, not the spread you will
see.** Humidity, wind, and microclimate wander around their means and drift back
over several days, which makes the observed standard deviation about 1.6 times the
authored value. Tune by looking at output, not by reading the number off the
curve. [TDD](Docs/TDD.md) has the arithmetic.

**Some inputs are authored but not yet used, and some channels are placeholders.**
`TemperatureStdDev_C` and a front's `LightningProbability` have no effect.
Temperature's daily swing uses a fixed trough at 06:00 and peak at 15:00 and does
not yet respond to generated humidity. Wind has sustained speed and front boosts
but no gusts. [TDD](Docs/TDD.md) lists them with the reasoning.

## Reference

### `FWeatherInitParams`

Passed to `Initialize` after the template. Every field is optional. In C++ use
designated initializers in declaration order and skip any you don't need.

| Field | Default | Meaning |
|---|---|---|
| `Seed` | `0` | `0` randomizes. Any other value is fully deterministic. |
| `Latitude` | `0` | Degrees north, negative for south. Clamped to `[-89.9, 89.9]`. |
| `StartingDay` | `1` | First `CurrentDay`, 1-based. Not validated. |
| `YearLength` | `365` | Days per year for climate-curve wraparound. Not validated. |
| `HorizonObstructionDeg` | `0` | Terrain angle hiding the sun. Shortens `DayLength_hours` only. Clamped to `[0, 89.9]`. |
| `AxialTiltDeg` | `23.45` | Planet's axial tilt. `0` gives no seasons; larger swings harder. Clamped to `[0, 90]`. |

The clamped values are what the accessors report (`GetLatitude`,
`GetHorizonObstructionDeg`, `GetAxialTiltDeg`); `GetSeed` reports the seed in use.

### `FWeatherDay` channels

| Channel | Type | Unit | Notes |
|---|---|---|---|
| `Day` | `int32` | | The day this is, 1-based. |
| `Temperature_C` | curve | °C | Seasonal mean plus a carried random deviation plus a diurnal wave. |
| `RelativeHumidity` | curve | fraction `[0, 1]` | Mean plus a carried deviation, clamped. No diurnal wave. |
| `WindSpeed_ms` | curve | m/s | Mean plus a carried deviation, plus a front's `WindBoost_ms` while one is active. Never negative. |
| `Rainfall_mm` | curve | mm/hour, water equivalent | Microclimate plus front, when at or above the snow threshold. |
| `Snowfall_mm` | curve | mm/hour, water equivalent | The same total, when below the threshold. Mutually exclusive with rainfall. |
| `SnowDepthRate_cm` | curve | cm/hour | Depth added by that hour's snowfall. A rate, not a running total. |
| `IsBlizzard` | curve | `1.0` / `0.0` | Snowing, front-driven, and at least 70% (currently) of that front's own peak intensity. |
| `SolarElevationDeg` | curve | degrees | True geometric elevation. Negative at night. |
| `SolarInsolation_Wm2` | curve | W/m² | Clear-sky irradiance reduced by that hour's rain, snow, and humidity. Zero at night. |
| `DayLength_hours` | `float` | hours | Accounts for `HorizonObstructionDeg`. `0` in polar night, `24` in polar day, never NaN. |

### `UWeatherTemplate` fields

| Field | Type | Default | Meaning |
|---|---|---|---|
| `MeanTemperature_C` | curve by day-of-year | empty | Seasonal temperature shape. |
| `TemperatureStdDev_C` | curve by day-of-year | empty | **Not yet read.** Temperature's random wander uses a fixed 1.5 °C scale. |
| `MeanRelativeHumidity`, `RelativeHumidityStdDev` | curves by day-of-year | empty | Seasonal humidity shape and spread. |
| `MeanWindSpeed_ms`, `WindStdDev_ms` | curves by day-of-year | empty | Sustained wind shape and spread. |
| `MicroclimateTypicalDays` | array of `FWeatherTypicalDay` | empty | Authored example days. Empty gives a dry climate. |
| `MicroclimateIntensityStdDev_mmPerHour` | float | `0` | Noise on top of the blended typical day. Not seasonal. |
| `FrontFrequency` | curve by day-of-year | empty | Daily likelihood that a front begins. |
| `FrontMeanDurationHours`, `FrontDurationStdDevHours` | float | `6`, `3` | How long a front lasts. Independent of which shape is drawn. |
| `TypicalFronts` | array of `FWeatherTypicalFront` | empty | Authored example passages. |
| `SnowTemperatureThresholdC` | float | `0` | Below this, precipitation is snow. |

`FWeatherTypicalDay` holds `IntensityByHourOfDay` and `SeasonalLikelihood`.
`FWeatherTypicalFront` holds `IntensityByPassageFraction`, `SeasonalLikelihood`,
`WindBoost_ms`, and `LightningProbability` (not yet read). Likelihood curves are
weights, not probabilities: they need not sum to 1 and are normalized at generation
time. All day-of-year curves are keyed on zero-based day-of-year `[0, 365]`.
