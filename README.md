# Weather — Stochastic Weather Generation for Unreal Engine 5

The Weather plugin turns a designer-authored climate description into a stream
of daily weather values: temperature, rainfall, wind, day length, solar
insolation, and relative humidity. It has no dependencies on any other plugin
and is equally useful outside the agriculture stack for any system that wants
physically-motivated daily climate data.

## How it works

A `UWeatherTemplate` data asset encodes the statistical envelope of a locale —
the seasonal shape and year-to-year variability of each channel. A
`UWeatherInstance` holds a seed and a cursor; each call to `AdvanceTo` draws
the next span of weather from those statistics and returns an `FWeatherSample`
carrying one curve per channel.

`UWeatherBlueprintLibrary` translates `FWeatherSample` into the input structs
that Soil and Vegetation consume. The core plugin — template, instance, sample
— knows nothing about those consumers.

## Docs layout

```
mainpage.md       Doxygen entry point — primary path, template authoring, gotchas
Docs/TDD.md       Technical design — stochastic models, determinism, test surface
Docs/UserGuide.md Designer-facing guide — authoring templates, tuning climate feel
```

## Companion repositories

| Repo | Contents |
|---|---|
| Weather_Code (private) | C++ source |
| Flora (public) | Docs for the Flora plant-simulation plugins |
| Flora_Code (private) | C++ source for Flora |
