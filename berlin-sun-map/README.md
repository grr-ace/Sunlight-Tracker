# Berlin Sun Map

Berlin Sun Map is an open-source web map for finding sunny places to walk,
sit, and spend time outdoors in Berlin. It combines a 3D map with the current
and upcoming position of the sun to explore how sunlight moves through the
city.

The project is an early prototype. Its goal is to make street-level sunlight
easier to understand, while gradually improving the model from directional
estimates to detailed building and tree shadows.

## Current features

- Berlin-focused MapLibre map using [OpenFreeMap](https://openfreemap.org/).
- 3D building extrusion from the map's building vector tiles.
- Street names without points-of-interest or parking symbols.
- Current sun altitude and compass direction calculated with
  [SunCalc](https://github.com/mourner/suncalc).
- Current sun direction and the next six hours shown on the map.
- Approximate building shadows based on visible building footprints, heights,
  sun altitude, and sun direction.
- Map rotation, pitch, zoom, and pan controls.

## How the current shadow model works

The prototype uses building footprints and height attributes available in the
visible OpenFreeMap vector tiles. For each building, it estimates the shadow
length using:

```text
shadow length = building height / tan(sun altitude)
```

It then projects the footprint away from the sun and draws the result as a
semi-transparent overlay. This is useful for exploring the concept, but it is
not yet a precise street-level exposure model. It does not currently account
for detailed roof geometry, occlusion between buildings, or trees.

## Development

### Requirements

- Node.js 22.18 or newer, or Node.js 24.12 or newer.
- An internet connection for OpenFreeMap styles and vector tiles.

### Setup

```sh
cd berlin-sun-map
npm install
npm run dev
```

Open the local URL printed by Vite, normally
`http://localhost:5173/`.

### Validation

```sh
npm run type-check
npm run build
```

## Technology

- [Vue 3](https://vuejs.org/)
- [TypeScript](https://www.typescriptlang.org/)
- [Vite](https://vite.dev/)
- [MapLibre GL JS](https://maplibre.org/)
- [OpenFreeMap](https://openfreemap.org/)
- [SunCalc](https://github.com/mourner/suncalc)

## Roadmap

### Sunlight controls

- Add a time slider and date picker for exploring the day.
- Show sunrise, sunset, golden hour, and nighttime states.
- Compare sunlight conditions at several times or across a full day.
- Let users choose a location in Berlin and inspect its sun exposure.

### More accurate buildings and shadows

- Integrate Berlin's official
  [LoD2 building models](https://www.berlin.de/sen/sbw/planung/geoinformation/).
- Build a preprocessing pipeline for Geoportal Berlin/FIS-Broker data.
- Convert the detailed dataset into efficient vector tiles, potentially using
  [Tippecanoe](https://github.com/mapbox/tippecanoe).
- Use more accurate building footprints, heights, wall surfaces, and roof
  shapes.
- Fine-tune shadow geometry for sloped roofs, roof ridges, setbacks, and
  elevated building sections.
- Account for buildings blocking sunlight from other buildings.
- Render a clearer sunlit/shaded suitability layer for streets, courtyards,
  parks, and public spaces.

### Trees and urban features

- Add Berlin tree data where open datasets and licensing permit.
- Model tree crowns and approximate seasonal shadow coverage.
- Account for tree height, canopy size, and deciduous versus evergreen trees.
- Add other relevant obstacles such as awnings, bridges, and large structures.

### User experience

- Add explanations and legends for sun direction, altitude, and shadows.
- Improve mobile touch controls and responsive layout.
- Add search and place lookup for Berlin streets and landmarks.
- Let users save or share a location and time.
- Add accessibility improvements, including color-safe overlays and keyboard
  controls.

## Data and accuracy

Map data, building data, and sunlight calculations have different sources and
different levels of accuracy. The current map should be treated as an
exploratory visualization rather than a guarantee that a particular street or
seat will be sunny.

Future releases will document dataset versions, processing steps, attribution,
and known limitations more precisely as additional official Berlin data is
integrated.