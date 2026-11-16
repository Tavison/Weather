# Weather: Stochastic Weather Generation for Unreal Engine 5

Weather generates believable daily weather from a climate you author. Describe
a place once as a `UWeatherTemplate` data asset, bind it to a `UWeatherInstance`,
and each `AdvanceDay` call returns the next day's hourly curves: temperature,
humidity, wind, rain, snow, blizzard flag, sun elevation and irradiance, plus the
day's length in hours.

The plugin holds one day at a time and has no clock. Rain comes from two
independent mechanisms, a persistent local microclimate and passing fronts, and
snow is a temperature gate on whatever is falling. Day length and irradiance are
computed from latitude, axial tilt, and horizon obstruction, so one template
works at any latitude and on any planet.

Among the project's plugins it depends only on `EcologyCommon` and
`STD_Utilities`. It knows nothing about Soil, Vegetation, or Botany, and is equally
useful for driving visual effects or ambience.

## Docs layout

```
mainpage.md   Doxygen entry point: the call sequence, gotchas, and field reference
Docs/TDD.md   Technical design: how a day is generated, why, and known limitations
```

The designer-facing guide to authoring a climate template is maintained in
Confluence, outside this repository.

## Companion repositories

| Repo | Contents |
|---|---|
| Weather (public) | These docs |
| Weather_Code (private) | C++ source for this plugin |
| EcologyCommon (private) | `FWeatherDay` and `IWeatherDaySource`, the contract Weather implements |
| WeatherGameplay (private) | World subsystem that owns a `UWeatherInstance` for gameplay code |
| WeatherGameplay_Docs (public) | Docs for WeatherGameplay |
| Flora (private) | Plant-simulation plugins, including Phenology, which consumes Weather |
| Flora_Docs (public) | Docs for the Flora plugins |
