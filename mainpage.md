\mainpage Weather Plugin

Stochastic daily weather generation for Unreal Engine 5.

## Overview

The Weather plugin turns a designer-authored climate description into a
stream of daily weather values: temperature, rainfall, wind, day length, and
solar insolation. A `UWeatherTemplate` encodes the statistics of a locale
— the seasonal shape and year-to-year variability. A `UWeatherInstance`
holds a seed and a cursor, and each call to `AdvanceTo` draws the next span
of weather from those statistics. The output is an `FWeatherSample` —
one curve per channel over the requested span — which `UWeatherBlueprintLibrary`
translates into the structs that Soil and Vegetation consume.

The plugin has no dependencies on Soil, Vegetation, or Botany; it is equally
useful outside the agriculture stack for any system that wants
physically-motivated daily climate data.

## The primary path

```cpp
// 1. Author a UWeatherTemplate data asset in the editor (one per locale).
//    Bind it at runtime:
UWeatherInstance* Weather = NewObject<UWeatherInstance>(Outer);
Weather->Initialize(WeatherTemplate, Seed, 51.5f);  // latitude in degrees north

// 2. Advance the cursor to produce weather curves for a span
FWeatherSample Sample = Weather->AdvanceTo(NowTime);
// Sample covers [previous LastProcessedTime, NowTime).
// After this call, Weather->GetLastProcessedTime() == NowTime.

// 3. Translate for Soil (curve path)
FSoilWeather SoilWeather = UWeatherBlueprintLibrary::WeatherSampleToSoilWeather(Sample);

// 4. Translate for Vegetation (curve path) — then fill the three soil-derived channels
FVegetationAmbient Ambient = UWeatherBlueprintLibrary::WeatherSampleToVegetationAmbient(Sample);
// SoilTemperature, SoilMoisture, and PAR are left empty by this call —
// fill them from your soil instance before passing to BotanyField::AdvanceTo.
```

Sequential `AdvanceTo` calls carry AR(1) state forward — temperature, wind, and relative humidity
are bit-exactly continuous at every span boundary. Advance week by week, day
by day, or in any span size; the output is always identical to what a single
call covering the full range would have produced.

## Tick path variant

When advancing simulation per game tick rather than per span, sample the
pre-computed `FWeatherSample` at a point in time:

```cpp
// Pre-compute a season once (or advance week by week)
FWeatherSample SeasonSample = Weather->AdvanceTo(SeasonEnd);

// Each game tick
FSoilWeatherInput SoilInput =
    UWeatherBlueprintLibrary::WeatherSampleToSoilWeatherInput(SeasonSample, NowTime, TickDays);

FVegetationTickAmbient TickAmbient =
    UWeatherBlueprintLibrary::WeatherSampleToVegetationTickAmbient(SeasonSample, NowTime);
// SoilTemperature, SoilMoisture, PAR left at zero — fill from soil state.
```

Weather curves are authored at daily resolution. All ticks within the same
calendar day evaluate to the same values; there is no intra-day variation
unless you supply higher-resolution curve data.

## Authoring a climate template

A `UWeatherTemplate` has twelve fields, all `FRuntimeFloatCurve` keyed on
day-of-year `[0, 365]`.

**Temperature** — `MeanTemperature_C`, `TemperatureStdDev_C`, `TemperatureAR1Alpha`

The mean curve shapes the seasonal cycle. The σ curve controls how variable
individual days are around that mean — winter days typically have wider σ than
summer. `TemperatureAR1Alpha` (default 0.6) governs cold-snap persistence:
higher values give each day more memory of the previous day.

**Rainfall** — `RainfallProbability`, `RainfallMean_mm`, `RainfallStdDev_mm`

Rainfall is per-day independent (not AR(1)). `RainfallProbability` is a
Bernoulli draw determining whether a day is wet; `RainfallMean_mm` and
`RainfallStdDev_mm` determine how much falls on wet days. Annual total ≈
`∫ probability(d) × mean_mm(d) dd`. Multi-day storm systems are a deferred
extension.

**Wind** — `MeanWindSpeed_ms`, `WindStdDev_ms`, `WindAR1Alpha`

Same model as temperature. `WindAR1Alpha` (default 0.4) is lower because
wind turns faster than temperature in most climates.

**Relative humidity** — `MeanRelativeHumidity`, `RelativeHumidityStdDev`, `RelativeHumidityAR1Alpha`

Same AR(1) model as temperature. Values are fractions in `[0, 1]` (0.75 = 75% RH);
draws are clamped to that range after each step. Used downstream to compute
vapour pressure deficit: `VPD = SVP(T) × (1 − RH)`. Typical temperate means
are 0.65–0.85, higher in winter, lower in summer. `RelativeHumidityAR1Alpha`
defaults to 0.6 — humid or dry air masses persist for several days.

**Day length and solar insolation are not on the template.** Both are
closed-form from latitude and day-of-year — the runtime computes them from
the latitude you pass to `Initialize`.

## Multiple locales and deterministic replay

```cpp
// Two independent locales from the same seed
WeatherA->Initialize(NorthernTemplate, Seed, 57.5f);
WeatherB->Initialize(SouthernTemplate, Seed, 51.5f);
```

Replay the same weather by saving `(TemplatePath, Seed, LatitudeDeg,
LastProcessedTime)` before an advance. Same four values → identical output.

## Gotchas

**`SetLastProcessedTime` breaks AR(1) continuity.** Repositioning the cursor
clears the carry. The next `AdvanceTo` uses a 30-day burn-in, producing a
statistical seam. For seamless continuity, advance sequentially without
repositioning.

**`WeatherSampleToVegetationAmbient` fills four channels and leaves three empty.**
It fills `AirTemperature`, `DayLength`, `Wind`, and `VPD_kPa`; `SoilTemperature`,
`SoilMoisture`, and `PAR` are left empty. Empty curves evaluate to effectively
zero in Vegetation, driving maximum stress. Always fill those three from soil
state before passing the ambient to Botany.

`VPD_kPa` is computed per key as `SVP(T) × (1 − RH)` where
`SVP(T) = 0.6108 × exp(17.27 × T / (T + 237.3))` kPa. The tick path
(`WeatherSampleToVegetationTickAmbient`) uses the same formula evaluated at
a single point in time.

**`RainfallMM` in the tick path is a total, not a rate.** `WeatherSampleToSoilWeatherInput`
multiplies the daily rate by `TickDays` before writing `FSoilWeatherInput.RainfallMM`.
Temperature, wind, and insolation are point-in-time values.

## Reference

### FWeatherSample channels

| Channel | Unit | Model |
|---|---|---|
| `AirTemperature_C` | °C | AR(1) from mean + σ |
| `Rainfall_mm_per_day` | mm/day | Bernoulli × Normal, i.i.d. per day |
| `WindSpeed_ms` | m/s | AR(1) from mean + σ, clamped ≥ 0 |
| `DayLength_hours` | hours | Closed-form from latitude + day-of-year |
| `SolarInsolation` | MJ/m²/day | Closed-form, attenuated by cloud-cover proxy |
| `RelativeHumidity` | fraction [0,1] | AR(1) from mean + σ, clamped to [0, 1] |

All keys placed at day midpoints (integer day + 0.5) on the game-time axis.

### Translation helpers

| Function | Output | Unfilled fields |
|---|---|---|
| `WeatherSampleToSoilWeather(Sample)` | `FSoilWeather` | — |
| `WeatherSampleToVegetationAmbient(Sample)` | `FVegetationAmbient` | SoilTemperature, SoilMoisture, PAR |
| `WeatherSampleToSoilWeatherInput(Sample, T, TickDays)` | `FSoilWeatherInput` | — |
| `WeatherSampleToVegetationTickAmbient(Sample, T)` | `FVegetationTickAmbient` | SoilTemperature, SoilMoisture, PAR |
