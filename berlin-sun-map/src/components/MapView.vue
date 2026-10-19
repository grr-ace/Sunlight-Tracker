<script setup lang="ts">
import { computed, onMounted, onBeforeUnmount, ref } from 'vue'
import * as maplibregl from 'maplibre-gl'
import * as SunCalc from 'suncalc'
import type { Feature, FeatureCollection, Geometry, MultiPolygon, Polygon, Position } from 'geojson'
import maplibreWorkerUrl from 'maplibre-gl/dist/maplibre-gl-worker.mjs?url'
import 'maplibre-gl/dist/maplibre-gl.css'

maplibregl.setWorkerUrl(maplibreWorkerUrl)

const mapContainer = ref<HTMLDivElement | null>(null)
const currentTime = ref(new Date())
const sunAltitude = computed(() => {
  const position = SunCalc.getPosition(currentTime.value, 52.52, 13.405)
  return Math.round(position.altitude)
})
const sunBearing = computed(() => {
  const position = SunCalc.getPosition(currentTime.value, 52.52, 13.405)
  return Math.round(position.azimuth)
})

// Plain variable, not ref()/reactive(): Vue's proxies can interfere with the map object
let map: maplibregl.Map | null = null
let clockTimer: number | undefined

const berlinCenter: [number, number] = [13.405, 52.52]

function destination(
  [longitude, latitude]: [number, number],
  bearing: number,
  distanceKm: number,
): [number, number] {
  const earthRadiusKm = 6371
  const angularDistance = distanceKm / earthRadiusKm
  const bearingRadians = (bearing * Math.PI) / 180
  const latitudeRadians = (latitude * Math.PI) / 180
  const longitudeRadians = (longitude * Math.PI) / 180
  const destinationLatitude = Math.asin(
    Math.sin(latitudeRadians) * Math.cos(angularDistance) +
      Math.cos(latitudeRadians) * Math.sin(angularDistance) * Math.cos(bearingRadians),
  )
  const destinationLongitude =
    longitudeRadians +
    Math.atan2(
      Math.sin(bearingRadians) * Math.sin(angularDistance) * Math.cos(latitudeRadians),
      Math.cos(angularDistance) - Math.sin(latitudeRadians) * Math.sin(destinationLatitude),
    )

  return [(destinationLongitude * 180) / Math.PI, (destinationLatitude * 180) / Math.PI]
}

function sunFeatures(date: Date): FeatureCollection<Geometry> {
  const position = SunCalc.getPosition(date, berlinCenter[1], berlinCenter[0])
  const bearing = position.azimuth
  const sunPoint = destination(berlinCenter, bearing, 2)
  const shadowPoint = destination(berlinCenter, (bearing + 180) % 360, 1.2)
  const nextFewHours = Array.from({ length: 7 }, (_, index) => {
    const futureDate = new Date(date.getTime() + index * 60 * 60 * 1000)
    const futurePosition = SunCalc.getPosition(futureDate, berlinCenter[1], berlinCenter[0])
    const futureBearing = futurePosition.azimuth
    return {
      type: 'Feature' as const,
      properties: { hour: futureDate.toISOString(), altitude: futurePosition.altitude },
      geometry: { type: 'Point' as const, coordinates: destination(berlinCenter, futureBearing, 1.7) },
    }
  })

  return {
    type: 'FeatureCollection',
    features: [
      {
        type: 'Feature',
        properties: { kind: 'sun-direction' },
        geometry: { type: 'LineString', coordinates: [berlinCenter, sunPoint] },
      },
      {
        type: 'Feature',
        properties: { kind: 'shadow-direction' },
        geometry: { type: 'LineString', coordinates: [berlinCenter, shadowPoint] },
      },
      {
        type: 'Feature',
        properties: { kind: 'sun-position', altitude: position.altitude },
        geometry: { type: 'Point', coordinates: sunPoint },
      },
      ...nextFewHours,
    ],
  }
}

function offsetPosition(position: Position, bearing: number, distanceMeters: number): Position {
  const longitude = position[0]
  const latitude = position[1]
  if (longitude === undefined || latitude === undefined) return position

  const latitudeRadians = (latitude * Math.PI) / 180
  const bearingRadians = (bearing * Math.PI) / 180
  const northMeters = Math.cos(bearingRadians) * distanceMeters
  const eastMeters = Math.sin(bearingRadians) * distanceMeters

  return [
    longitude + eastMeters / (111_320 * Math.cos(latitudeRadians)),
    latitude + northMeters / 110_574,
  ]
}

function offsetGeometry(
  geometry: Polygon | MultiPolygon,
  bearing: number,
  distanceMeters: number,
): Polygon | MultiPolygon {
  const offsetRing = (ring: Position[]) =>
    ring.map((position) => offsetPosition(position, bearing, distanceMeters))

  if (geometry.type === 'Polygon') {
    return { type: 'Polygon', coordinates: geometry.coordinates.map(offsetRing) }
  }

  return {
    type: 'MultiPolygon',
    coordinates: geometry.coordinates.map((polygon) => polygon.map(offsetRing)),
  }
}

function updateBuildingShadows() {
  if (!map?.getLayer('berlin-building-shadows')) return

  const source = map.getSource('building-shadows') as maplibregl.GeoJSONSource | undefined
  if (!source) return

  const position = SunCalc.getPosition(currentTime.value, berlinCenter[1], berlinCenter[0])
  if (position.altitude <= 0) {
    source.setData({ type: 'FeatureCollection', features: [] })
    return
  }

  const shadowBearing = (position.azimuth + 180) % 360
  const features: Feature<Polygon | MultiPolygon>[] = []
  const seen = new Set<string>()

  map.queryRenderedFeatures({ layers: ['berlin-buildings-3d'] }).forEach((building) => {
    if (building.geometry.type !== 'Polygon' && building.geometry.type !== 'MultiPolygon') return

    const geometryKey = JSON.stringify(building.geometry)
    if (seen.has(geometryKey)) return
    seen.add(geometryKey)

    const properties = building.properties ?? {}
    const height = Number(properties.render_height ?? properties.height ?? 8)
    const baseHeight = Number(properties.render_min_height ?? properties.min_height ?? 0)
    const effectiveHeight = Math.max(height - baseHeight, 1)
    const shadowLength = Math.min(effectiveHeight / Math.tan((position.altitude * Math.PI) / 180), 500)

    features.push({
      type: 'Feature',
      properties: { height: effectiveHeight },
      geometry: offsetGeometry(building.geometry, shadowBearing, shadowLength),
    })
  })

  source.setData({ type: 'FeatureCollection', features })
}

onMounted(() => {
  if (!mapContainer.value) return

  map = new maplibregl.Map({
    container: mapContainer.value,
    style: 'https://tiles.openfreemap.org/styles/liberty',
    center: [13.405, 52.52], // [longitude, latitude]: Berlin Mitte
    zoom: 16,
    pitch: 60,
    bearing: -20,
    maxBounds: [
      [13.0, 52.3], // south-west corner
      [13.85, 52.72], // north-east corner
    ],
  })

  map.addControl(new maplibregl.NavigationControl())

  map.once('load', () => {
    const style = map?.getStyle()
    const firstSymbolLayer = style?.layers?.find((layer) => layer.type === 'symbol')

    style?.layers
      .filter((layer) => layer.type === 'symbol')
      .forEach((layer) => {
        const isStreetName = layer.id.startsWith('highway-name-')
        map?.setLayoutProperty(layer.id, 'visibility', isStreetName ? 'visible' : 'none')
      })

    if (map && !map.getLayer('berlin-buildings-3d')) {
      map.addLayer(
        {
          id: 'berlin-buildings-3d',
          source: 'openmaptiles',
          'source-layer': 'building',
          type: 'fill-extrusion',
          minzoom: 14,
          paint: {
            'fill-extrusion-color': '#b7c4d3',
            'fill-extrusion-height': ['coalesce', ['get', 'render_height'], ['get', 'height'], 8],
            'fill-extrusion-base': ['coalesce', ['get', 'render_min_height'], ['get', 'min_height'], 0],
            'fill-extrusion-opacity': 0.72,
          },
        },
        firstSymbolLayer?.id,
      )
    }

    map?.addSource('building-shadows', {
      type: 'geojson',
      data: { type: 'FeatureCollection', features: [] },
    })
    map?.addLayer(
      {
        id: 'berlin-building-shadows',
        type: 'fill',
        source: 'building-shadows',
        paint: {
          'fill-color': '#4b426f',
          'fill-opacity': 0.34,
        },
      },
      'berlin-buildings-3d',
    )
    map?.on('moveend', updateBuildingShadows)
    map?.on('zoomend', updateBuildingShadows)
    requestAnimationFrame(updateBuildingShadows)

    map?.addSource('sunlight', {
      type: 'geojson',
      data: sunFeatures(currentTime.value),
    })
    map?.addLayer({
      id: 'sun-path',
      type: 'circle',
      source: 'sunlight',
      filter: ['==', ['get', 'kind'], 'sun-position'],
      paint: {
        'circle-color': '#ffd166',
        'circle-radius': 10,
        'circle-stroke-color': '#fff4bf',
        'circle-stroke-width': 2,
      },
    })
    map?.addLayer({
      id: 'sun-path-hours',
      type: 'circle',
      source: 'sunlight',
      filter: ['has', 'hour'],
      paint: {
        'circle-color': '#ffd166',
        'circle-opacity': 0.55,
        'circle-radius': 5,
      },
    })
    map?.addLayer({
      id: 'sun-direction',
      type: 'line',
      source: 'sunlight',
      filter: ['==', ['get', 'kind'], 'sun-direction'],
      paint: { 'line-color': '#ffd166', 'line-width': 3, 'line-opacity': 0.9 },
    })
    map?.addLayer({
      id: 'shadow-direction',
      type: 'line',
      source: 'sunlight',
      filter: ['==', ['get', 'kind'], 'shadow-direction'],
      paint: { 'line-color': '#6c63a8', 'line-width': 2, 'line-dasharray': [2, 2], 'line-opacity': 0.8 },
    })
  })

  clockTimer = window.setInterval(() => {
    currentTime.value = new Date()
    const source = map?.getSource('sunlight') as maplibregl.GeoJSONSource | undefined
    source?.setData(sunFeatures(currentTime.value))
    updateBuildingShadows()
  }, 60_000)
})

onBeforeUnmount(() => {
  if (clockTimer !== undefined) window.clearInterval(clockTimer)
  map?.off('moveend', updateBuildingShadows)
  map?.off('zoomend', updateBuildingShadows)
  map?.remove()
})
</script>

<template>
  <main class="map-shell">
    <div ref="mapContainer" class="map"></div>
    <section class="sun-panel" aria-live="polite">
      <h1>Berlin sunlight</h1>
      <p>Current sun direction and the next six hours</p>
      <dl>
        <div>
          <dt>Sun altitude</dt>
          <dd>{{ sunAltitude }}°</dd>
        </div>
        <div>
          <dt>Direction</dt>
          <dd>{{ sunBearing }}°</dd>
        </div>
      </dl>
      <small>Yellow: sunlight direction · Purple: approximate shadow direction</small>
    </section>
  </main>
</template>

<style scoped>
.map-shell,
.map {
  width: 100%;
  height: 100vh;
}

.map-shell {
  position: relative;
}

.sun-panel {
  position: absolute;
  top: 1rem;
  left: 1rem;
  z-index: 1;
  max-width: 19rem;
  padding: 1rem 1.1rem;
  border: 1px solid rgb(255 255 255 / 35%);
  border-radius: 0.75rem;
  color: #fff;
  background: rgb(20 29 43 / 88%);
  box-shadow: 0 0.4rem 1.5rem rgb(0 0 0 / 20%);
  backdrop-filter: blur(0.4rem);
}

h1 {
  font-size: 1.15rem;
  font-weight: 700;
}

p {
  margin: 0.2rem 0 0.9rem;
  color: #d5deeb;
  font-size: 0.85rem;
}

dl {
  display: flex;
  gap: 1.5rem;
}

dt,
small {
  color: #b8c4d3;
  font-size: 0.75rem;
}

dd {
  font-size: 1.2rem;
  font-weight: 700;
}

small {
  display: block;
  margin-top: 0.85rem;
  line-height: 1.35;
}
</style>