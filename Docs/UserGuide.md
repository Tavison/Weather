# Weather Plugin — User Guide

## Overview

The Weather plugin generates physically-motivated daily weather data from
authored climate statistics. Given a **UWeatherTemplate** (mean, variance, and
autocorrelation for temperature, rainfall, and wind) and a geographic latitude,
a **UWeatherInstance** acts as a cursor that advances through game time,
producing an **FWeatherSample** — a set of five curves for each span advanced.

| Channel | Unit | Generation method |
|---|---|---|
| Air temperature | °C | AR(1) autoregressive |
| Rainfall | mm/day | i.i.d. Bernoulli × Normal |
| Wind speed | m/s | AR(1) autoregressive |
| Day length | hours | Closed-form (latitude + day-of-year) |
| Solar insolation | normalized [0, 1] | Closed-form, attenuated by rain probability |

The output is translated into the data structures expected by the **Soil** and
**Vegetation** plugins via **UWeatherBlueprintLibrary**, making Weather the
environmental input layer for the Botany demand/supply pipeline.

The plugin ships with two simulation paths:

- **Curve path** — generate a full span of weather up-front; hand curves directly
  to `UBotanyField::AdvanceTo`. Preferred for turn-based or forward-simulation
  workflows.
- **Tick path** — sample a single scalar snapshot per game tick; hand it to
  `UBotanyField::TickField`. Preferred for real-time or game-loop-driven workflows.

---

## Concepts

### Climate vs. Weather

A **UWeatherTemplate** encodes **climate** — the statistical distribution of
weather conditions across the year. Each field is a curve keyed on day-of-year
[0, 365]. The template itself never changes at runtime.

A **UWeatherInstance** generates **weather** — a specific stochastic realisation
drawn from those distributions for a requested span of game-time days. Two
instances with different seeds, or with the same seed but different latitudes,
will produce different sequences.

### AR(1) Autocorrelation

Temperature and wind use a first-order autoregressive (AR(1)) process to
produce realistic day-to-day persistence:

```
Today = α · Yesterday + (1 − α) · Mean(day) + Normal(0, σ(day))
```

- **α near 1** — slow-changing (high persistence); temperature swings are gradual.
- **α near 0** — each day is nearly independent of the previous one.
- **α = 0** — fully i.i.d.; no memory between days.

`TemperatureAR1Alpha` defaults to 0.6. `WindAR1Alpha` defaults to 0.4.

### Rainfall

Rainfall is **not** AR(1). Each day is an independent two-stage draw:

1. Bernoulli trial against `RainfallProbability(day)` — is it a wet day?
2. On wet days, amount = `Normal(RainfallMean_mm, RainfallStdDev_mm)`, clamped ≥ 0.

### Day Length and Solar Insolation

Both are deterministic and require no authoring beyond the latitude supplied to
`Initialize`:

- **Day length** is computed from latitude and day-of-year using the standard
  declination formula.
- **Solar insolation** is derived from day length, attenuated by a cloud-cover
  proxy built from `RainfallProbability`.

Neither field appears in UWeatherTemplate.

### Determinism

`AdvanceTo` is reproducible: the same cursor position, seed, template, and
latitude always produce the same output. Sequential advances carry state
forward and are bit-exactly continuous. After `SetLastProcessedTime` or
`Initialize`, the next `AdvanceTo` uses a 30-day burn-in and is reproducible
as a pure function of `(Template, Seed, LatitudeDeg, LastProcessedTime, T_end)`.

### AR(1) Carry State and Span Continuity

`UWeatherInstance` keeps a lightweight carry buffer: after each `AdvanceTo`
call it stores the final temperature and wind AR(1) state. Sequential calls
forward that carry directly — temperature and wind are bit-exactly continuous
at every span boundary:

```
AdvanceTo(7)   →  carry stored at T=7;  LastProcessedTime = 7
AdvanceTo(14)  →  carry forwarded; days 7–13 are bit-identical to
                  what a single AdvanceTo(14) from 0 would have produced
```

Repositioning via `SetLastProcessedTime` clears the carry. The next `AdvanceTo`
falls back to a 30-day burn-in and will have a small statistical seam relative
to a continuously-advanced sequence.

> **Rule:** For seamless continuity, advance sequentially without repositioning.
> After `SetLastProcessedTime`, the next span uses burn-in and is approximately
> (not bit-exactly) continuous with any prior sequence.

### Latitude

Latitude is supplied in **degrees north**. Negative values are south of the
equator. The value is clamped to [−89.9, 89.9] internally to avoid polar
singularities in the day-length formula.

---

## System Anatomy

### UWeatherTemplate

A `UDataAsset` subclass. All fields are `FRuntimeFloatCurve` curves keyed on
**day-of-year** (X-axis: 0 = January 1, 364 = December 31).

| Field | Description |
|---|---|
| `MeanTemperature_C` | Daily mean air temperature |
| `TemperatureStdDev_C` | Daily temperature standard deviation (≥ 0) |
| `TemperatureAR1Alpha` | AR(1) smoothing coefficient, [0, 1] |
| `RainfallProbability` | Probability of a wet day, [0, 1] |
| `RainfallMean_mm` | Mean rainfall on wet days (≥ 0) |
| `RainfallStdDev_mm` | Rainfall standard deviation on wet days (≥ 0) |
| `MeanWindSpeed_ms` | Daily mean wind speed (≥ 0) |
| `WindStdDev_ms` | Daily wind standard deviation (≥ 0) |
| `WindAR1Alpha` | AR(1) smoothing coefficient for wind, [0, 1] |

### UWeatherInstance

A `UObject` subclass. Acts as a cursor over the climate template.

| Method | Description |
|---|---|
| `Initialize(Template, Seed, LatitudeDeg)` | Bind the template, seed, and latitude. Sets `LastProcessedTime = 0`. Must be called before `AdvanceTo`. |
| `AdvanceTo(T_end)` | Advance from `LastProcessedTime` to `T_end`. Returns `FWeatherSample`. Updates `LastProcessedTime`. |
| `GetLastProcessedTime()` | Returns the cursor position — the exclusive lower bound of the next `AdvanceTo` span. |
| `SetLastProcessedTime(T)` | Reposition the cursor without computing the span. Clears carry; next `AdvanceTo` uses burn-in. |

### FWeatherSample

| Field | Type | Description |
|---|---|---|
| `AirTemperature_C` | `FRuntimeFloatCurve` | Daily air temperature in °C |
| `Rainfall_mm_per_day` | `FRuntimeFloatCurve` | Daily rainfall in mm |
| `WindSpeed_ms` | `FRuntimeFloatCurve` | Daily wind speed in m/s |
| `DayLength_hours` | `FRuntimeFloatCurve` | Daylight hours |
| `SolarInsolation` | `FRuntimeFloatCurve` | Normalised solar energy [0, 1] |
| `SpanStart` | `double` | First day of the sampled span |
| `SpanEnd` | `double` | One past the last day of the sampled span |

All curves are keyed at day midpoints (DayIndex + 0.5 on the X-axis).

### UWeatherBlueprintLibrary

Static helper functions for translating `FWeatherSample` into the structs
expected by the Soil and Vegetation plugins.

**Curve path (AdvanceTo workflow):**

| Function | Output | Notes |
|---|---|---|
| `WeatherSampleToSoilWeather(Sample)` | `FSoilWeather` | Copies AirTemperature_C, Rainfall_mm_per_day, WindSpeed_ms, SolarInsolation curves |
| `WeatherSampleToVegetationAmbient(Sample)` | `FVegetationAmbient` | Copies AirTemperature_C, DayLength_hours, WindSpeed_ms curves; **SoilTemperature, SoilMoisture, PAR left empty** |

**Tick path (TickField workflow):**

| Function | Output | Notes |
|---|---|---|
| `WeatherSampleToSoilWeatherInput(Sample, T, TickDays)` | `FSoilWeatherInput` | Point-in-time sample at T; RainfallMM is scaled by TickDays |
| `WeatherSampleToVegetationTickAmbient(Sample, T)` | `FVegetationTickAmbient` | Point-in-time scalars at T; SoilTemperature, SoilMoisture, PAR left at zero |

---

## Prerequisites

- **Soil plugin** — required by UWeatherBlueprintLibrary for `FSoilWeather` /
  `FSoilWeatherInput` types.
- **Vegetation plugin** — required by UWeatherBlueprintLibrary for
  `FVegetationAmbient` / `FVegetationTickAmbient` types.
- **Botany plugin** — required to drive `UBotanyField::AdvanceTo` or `TickField`
  with the translated weather structs.

The Weather plugin itself has no runtime dependencies on the above plugins; only
`UWeatherBlueprintLibrary` (which lives in the Weather module) pulls them in.

---

## Step 1 — Author a UWeatherTemplate Data Asset

1. In the Content Browser, right-click → **Miscellaneous → Data Asset**.
2. Select `UWeatherTemplate` as the class.
3. Open the asset. For each of the nine curve fields, add keys across the
   day-of-year range [0, 365] to describe the annual climate cycle.

**Authoring rules:**

- All standard deviation fields (`TemperatureStdDev_C`, `RainfallStdDev_mm`,
  `WindStdDev_ms`) must be ≥ 0 everywhere. Negative values will be treated as
  zero internally but will not produce an error.
- AR(1) alpha fields (`TemperatureAR1Alpha`, `WindAR1Alpha`) should stay in
  [0, 1]. Values outside this range are not clamped and will produce
  non-stationary sequences.
- `RainfallProbability` must be in [0, 1] for the Bernoulli draw to be valid.
- A flat curve (single key at X = 0) is valid and produces a seasonally
  constant climate.

**Minimal viable template** — add at least one key to every curve, even if
flat. Curves with no keys evaluate to 0.0, which silently zeroes out the
corresponding weather channel.

---

## Step 2 — Create and Initialize a UWeatherInstance

```cpp
UWeatherInstance* Weather = NewObject<UWeatherInstance>(Outer);
Weather->Initialize(WeatherTemplate, Seed, LatitudeDeg);
```

| Parameter | Type | Notes |
|---|---|---|
| `WeatherTemplate` | `UWeatherTemplate*` | The data asset authored in Step 1 |
| `Seed` | `int32` | Any integer; same seed → same sequence |
| `LatitudeDeg` | `float` | Degrees north; negative = south; clamped to [−89.9, 89.9] |

`Initialize` must be called before `AdvanceTo`. The instance is safe to call
from a single thread; do not share across threads without external coordination.

---

## Step 3 — Advance Weather

```cpp
const FWeatherSample Sample = Weather->AdvanceTo(T_end);
```

`T_end` is a **game-time day** (the same unit used by `UBotanyField::AdvanceTo`).
The returned sample covers `[LastProcessedTime, T_end)`. After the call,
`Weather->GetLastProcessedTime() == T_end`.

To start sampling at a day other than 0 without computing the preceding span:

```cpp
Weather->SetLastProcessedTime(StartDay);
const FWeatherSample Sample = Weather->AdvanceTo(StartDay + SpanDays);
```

Sequential `AdvanceTo` calls carry the AR(1) state forward — temperature and
wind are bit-exactly continuous at every span boundary. After `SetLastProcessedTime`,
the next call uses a 30-day burn-in and will be approximately continuous.

The returned `FWeatherSample` is a value type — copy it freely. All curves
inside it are self-contained `FRuntimeFloatCurve` objects.

---

## Step 4 — Build FSoilWeather (Curve Path)

```cpp
const FSoilWeather SoilWeather =
    UWeatherBlueprintLibrary::WeatherSampleToSoilWeather(Sample);
```

This copies four channels from the sample directly into an `FSoilWeather`
struct. The channel mapping is:

| FWeatherSample channel | FSoilWeather field |
|---|---|
| `AirTemperature_C` | `AirTemperature_C` |
| `Rainfall_mm_per_day` | `Rainfall_mm_per_day` |
| `WindSpeed_ms` | `WindSpeed_ms` |
| `SolarInsolation` | `SolarInsolation` |

`DayLength_hours` is not included in `FSoilWeather`; it goes to
`FVegetationAmbient` in the next step.

---

## Step 5 — Build FVegetationAmbient (Curve Path)

```cpp
FVegetationAmbient VegAmbient =
    UWeatherBlueprintLibrary::WeatherSampleToVegetationAmbient(Sample);
```

This copies three atmospheric channels. The channel mapping is:

| FWeatherSample channel | FVegetationAmbient field |
|---|---|
| `AirTemperature_C` | `AirTemperature` |
| `DayLength_hours` | `DayLength` |
| `WindSpeed_ms` | `Wind` |

> **Required: fill the soil-derived channels before passing to AdvanceTo.**
>
> `WeatherSampleToVegetationAmbient` deliberately leaves three fields
> **empty** (no curve keys):
>
> - `SoilTemperature`
> - `SoilMoisture`
> - `PAR` (photosynthetically active radiation)
>
> These values come from Soil state, not from Weather. The Vegetation plugin
> does not silently treat empty curves as zero — curves with no keys evaluate
> to the engine default (usually 0.0), which will drive plants to zero stress
> response. Fill these fields from your `USoilInstance` before calling
> `AdvanceTo`.
>
> Even a flat constant curve (a single key at X = Sample.SpanStart with the
> current soil value) is valid. An empty curve is not.

Example — read from soil and install flat curves:

```cpp
const FSoilSample Soil = SoilInstance->GetSample();

FRichCurve* SoilTemp = VegAmbient.SoilTemperature.GetRichCurve();
SoilTemp->AddKey(Sample.SpanStart, Soil.Temperature_C);

FRichCurve* SoilMoisture = VegAmbient.SoilMoisture.GetRichCurve();
SoilMoisture->AddKey(Sample.SpanStart, Soil.VolumetricWaterContent);

FRichCurve* PAR = VegAmbient.PAR.GetRichCurve();
PAR->AddKey(Sample.SpanStart, Soil.PAR);  // or compute from insolation
```

If you are using `UBotanyField::AdvanceTo`, the field handles soil state
internally and constructs the correct ambient from its own soil reference —
you do not need to fill these fields manually.

---

## Step 6 — Advance the Field (Curve Path)

```cpp
Field->AdvanceTo(NowTime, SoilWeather, VegAmbient);
```

`UBotanyField::AdvanceTo` runs the full demand/supply pipeline for all plants
in the field from `LastProcessedTime` to `NowTime` using the supplied weather
and ambient curves.

After this call, query plant state via `UVegetationInstance::GetSample()` and
soil state via `USoilInstance::GetSample()`.

---

## Quick Reference

### UWeatherInstance API

| Call | Returns | Notes |
|---|---|---|
| `Initialize(Template, Seed, Lat)` | `void` | Must precede AdvanceTo; sets cursor to 0 |
| `AdvanceTo(T_end)` | `FWeatherSample` | Advances cursor; sequential calls bit-exact |
| `GetLastProcessedTime()` | `double` | Current cursor position |
| `SetLastProcessedTime(T)` | `void` | Reposition without computing; clears carry |

### UWeatherBlueprintLibrary — Curve Path

| Call | Returns | Soil-derived fields? |
|---|---|---|
| `WeatherSampleToSoilWeather(Sample)` | `FSoilWeather` | N/A |
| `WeatherSampleToVegetationAmbient(Sample)` | `FVegetationAmbient` | **Empty — caller must fill** |

### UWeatherBlueprintLibrary — Tick Path

| Call | Returns | Notes |
|---|---|---|
| `WeatherSampleToSoilWeatherInput(Sample, T, TickDays)` | `FSoilWeatherInput` | RainfallMM scaled by TickDays |
| `WeatherSampleToVegetationTickAmbient(Sample, T)` | `FVegetationTickAmbient` | SoilTemperature/SoilMoisture/PAR = 0 |

### Template Field Constraints

| Field | Constraint |
|---|---|
| `TemperatureStdDev_C` | ≥ 0 |
| `TemperatureAR1Alpha` | [0, 1] recommended |
| `RainfallProbability` | [0, 1] |
| `RainfallMean_mm` | ≥ 0 |
| `RainfallStdDev_mm` | ≥ 0 |
| `MeanWindSpeed_ms` | ≥ 0 |
| `WindStdDev_ms` | ≥ 0 |
| `WindAR1Alpha` | [0, 1] recommended |

---

## Extended Workflows

### Tick Path (Game-Loop Integration)

Use the tick path when you need to advance simulation one game tick at a time
rather than simulating a full span up-front.

The tick path still requires an `FWeatherSample`. You can either advance a full
season up-front, or advance week by week — sequential `AdvanceTo` calls carry
state forward so there is no seam at week boundaries:

```cpp
// At field initialization — advance a full season
FWeatherSample SeasonSample = WeatherInstance->AdvanceTo(SeasonEnd);

// Each game tick
const float TickDays = DeltaSeconds / SecondsPerDay;
const double NowTime = CurrentGameTimeDays;

const FSoilWeatherInput SoilInput =
    UWeatherBlueprintLibrary::WeatherSampleToSoilWeatherInput(
        SeasonSample, NowTime, TickDays);

FVegetationTickAmbient VegAmbient =
    UWeatherBlueprintLibrary::WeatherSampleToVegetationTickAmbient(
        SeasonSample, NowTime);

// Fill soil-derived channels from live soil state
const FSoilSample Soil = Field->GetSoil()->GetSample();
VegAmbient.SoilTemperature = Soil.Temperature_C;
VegAmbient.SoilMoisture    = Soil.VolumetricWaterContent;
VegAmbient.PAR             = ComputePAR(SeasonSample, NowTime);

Field->TickField(TickDays, SoilInput, VegAmbient);
```

> **SoilTemperature, SoilMoisture, and PAR in FVegetationTickAmbient are always
> zero after WeatherSampleToVegetationTickAmbient.** Fill them from live soil
> state before every TickField call — the same partial-fill requirement as the
> curve path.

`FSoilWeatherInput.RainfallMM` is automatically scaled by `TickDays` inside
`WeatherSampleToSoilWeatherInput`, so you do not need to scale it manually.

### Deterministic Seeding and Reproducibility

`AdvanceTo` is reproducible: saving `(TemplatePath, Seed, LatitudeDeg,
LastProcessedTime)` before a call is sufficient to replay the same sequence.

To generate different weather for the same climate template (e.g., two farms
in the same region that shouldn't be perfectly correlated), use distinct seeds:

```cpp
WeatherA->Initialize(Template, SeedA, Latitude);
WeatherB->Initialize(Template, SeedB, Latitude);
```

The sequences will be statistically similar (same climate) but sample-by-sample
independent.

### Multiple Locales

Each `UWeatherInstance` is independent. To simulate several geographic
locations simultaneously, create one instance per locale:

```cpp
WeatherHighland->Initialize(HighlandTemplate, Seed, 57.5f);   // Scottish highlands
WeatherLowland->Initialize(LowlandTemplate,  Seed, 51.5f);    // English lowlands
```

Different templates let you author distinct climate statistics; different
latitudes shift day length and insolation automatically.

### Extending the Span

Because `AdvanceTo` moves the cursor forward, extending an open-ended simulation
is simply another `AdvanceTo` call — no bookkeeping required:

```cpp
if (NowTime >= CurrentSample.SpanEnd)
{
    CurrentSample = WeatherInstance->AdvanceTo(CurrentSample.SpanEnd + ExtensionDays);
}
```

The cursor is already at `CurrentSample.SpanEnd` from the previous call, so the
carry is forwarded and the new span is bit-exactly continuous with the old one.
