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
const isSunPanelVisible = ref(true)
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
let slowRotateCleanup: (() => void) | undefined

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

function enableSlowControlRotation(mapInstance: maplibregl.Map, container: HTMLDivElement) {
  mapInstance.dragRotate.disable()
  mapInstance.dragPan.enable()

  let isRotating = false
  let lastX = 0
  let lastY = 0
  const sensitivity = 0.7
  const smoothing = 0.22
  let targetBearing = mapInstance.getBearing()
  let targetPitch = mapInstance.getPitch()
  let animationFrame: number | undefined

  const normalizeBearingDelta = (delta: number) => ((delta + 540) % 360) - 180

  const animateCamera = () => {
    const currentBearing = mapInstance.getBearing()
    const currentPitch = mapInstance.getPitch()
    const bearingDelta = normalizeBearingDelta(targetBearing - currentBearing)
    const pitchDelta = targetPitch - currentPitch

    mapInstance.setBearing(currentBearing + bearingDelta * smoothing)
    mapInstance.setPitch(currentPitch + pitchDelta * smoothing)

    if (isRotating || Math.abs(bearingDelta) > 0.01 || Math.abs(pitchDelta) > 0.01) {
      animationFrame = window.requestAnimationFrame(animateCamera)
    } else {
      animationFrame = undefined
    }
  }

  const startAnimation = () => {
    if (animationFrame === undefined) {
      animationFrame = window.requestAnimationFrame(animateCamera)
    }
  }

  const onContextMenu = (event: MouseEvent) => {
    if (isRotating) event.preventDefault()
  }

  const onPointerDown = (event: PointerEvent) => {
    if (!event.ctrlKey || event.button !== 0) return

    isRotating = true
    lastX = event.clientX
    lastY = event.clientY
    targetBearing = mapInstance.getBearing()
    targetPitch = mapInstance.getPitch()
    mapInstance.dragPan.disable()
    container.setPointerCapture(event.pointerId)
    startAnimation()
    event.preventDefault()
    event.stopPropagation()
  }

  const onPointerMove = (event: PointerEvent) => {
    if (!isRotating) return

    const deltaX = event.clientX - lastX
    const deltaY = event.clientY - lastY
    targetBearing += deltaX * sensitivity
    targetPitch -= deltaY * sensitivity
    lastX = event.clientX
    lastY = event.clientY
    startAnimation()
    event.preventDefault()
    event.stopPropagation()
  }

  const stopRotating = (event: PointerEvent) => {
    if (!isRotating) return

    isRotating = false
    targetBearing = mapInstance.getBearing()
    targetPitch = mapInstance.getPitch()
    if (animationFrame !== undefined) {
      window.cancelAnimationFrame(animationFrame)
      animationFrame = undefined
    }
    if (container.hasPointerCapture(event.pointerId)) {
      container.releasePointerCapture(event.pointerId)
    }
    mapInstance.dragPan.enable()
    event.stopPropagation()
  }

  const onKeyUp = (event: KeyboardEvent) => {
    if (event.key === 'Control' && isRotating) {
      isRotating = false
      targetBearing = mapInstance.getBearing()
      targetPitch = mapInstance.getPitch()
      if (animationFrame !== undefined) {
        window.cancelAnimationFrame(animationFrame)
        animationFrame = undefined
      }
      mapInstance.dragPan.enable()
    }
  }

  container.addEventListener('contextmenu', onContextMenu)
  container.addEventListener('pointerdown', onPointerDown, true)
  container.addEventListener('pointermove', onPointerMove, true)
  container.addEventListener('pointerup', stopRotating, true)
  container.addEventListener('pointercancel', stopRotating, true)
  window.addEventListener('keyup', onKeyUp)

  return () => {
    if (animationFrame !== undefined) window.cancelAnimationFrame(animationFrame)
    mapInstance.dragRotate.enable()
    mapInstance.dragPan.enable()
    container.removeEventListener('contextmenu', onContextMenu)
    container.removeEventListener('pointerdown', onPointerDown, true)
    container.removeEventListener('pointermove', onPointerMove, true)
    container.removeEventListener('pointerup', stopRotating, true)
    container.removeEventListener('pointercancel', stopRotating, true)
    window.removeEventListener('keyup', onKeyUp)
  }
}

onMounted(() => {
  if (!mapContainer.value) return

  map = new maplibregl.Map({
    container: mapContainer.value,
    style: 'https://tiles.openfreemap.org/styles/liberty',
    center: [13.405, 52.52], // [longitude, latitude]: Berlin Mitte
    zoom: 15.7,
    pitch: 80,
    bearing: 2,
    maxBounds: [
      [13.0, 52.3], // south-west corner
      [13.85, 52.72], // north-east corner
    ],
  })

  map.addControl(new maplibregl.NavigationControl())
  slowRotateCleanup = enableSlowControlRotation(map, mapContainer.value)

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
  slowRotateCleanup?.()
  map?.off('moveend', updateBuildingShadows)
  map?.off('zoomend', updateBuildingShadows)
  map?.remove()
})
</script>

<template>
  <main class="map-shell">
    <div ref="mapContainer" class="map"></div>
    <section class="sun-panel" :class="{ 'sun-panel-hidden': !isSunPanelVisible }" aria-live="polite">
      <div class="sun-panel-header">
        <h1>Berlin sunlight</h1>
        <button
          class="panel-toggle"
          type="button"
          :aria-expanded="isSunPanelVisible"
          :aria-label="isSunPanelVisible ? 'Hide sunlight details' : 'Show sunlight details'"
          @click="isSunPanelVisible = !isSunPanelVisible"
        >
          <span aria-hidden="true">‹</span>
        </button>
      </div>
      <template v-if="isSunPanelVisible">
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
      </template>
    </section>
    <button
      v-if="!isSunPanelVisible"
      class="sun-panel-reveal"
      type="button"
      aria-label="Show sunlight details"
      @click="isSunPanelVisible = true"
    >
      <span aria-hidden="true">›</span>
    </button>
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
  --sun-panel-color: rgb(20 29 43 / 88%);
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
  background: var(--sun-panel-color);
  box-shadow: 0 0.4rem 1.5rem rgb(0 0 0 / 20%);
  backdrop-filter: blur(0.4rem);
  transition: transform 0.2s ease;
}

.sun-panel-hidden {
  transform: translateX(calc(-100% - 1rem));
}

.sun-panel-header {
  display: flex;
  align-items: center;
  justify-content: flex-start;
  gap: 1rem;
}

.panel-toggle {
  display: grid;
  width: 1.6rem;
  height: 1.6rem;
  padding: 0;
  border: 1px solid rgb(255 255 255 / 40%);
  border-radius: 0.35rem;
  place-items: center;
  color: #fff;
  background: var(--sun-panel-color);
  cursor: pointer;
  font: inherit;
  font-size: 1.15rem;
  line-height: 1;
}

.sun-panel-header .panel-toggle {
  position: absolute;
  top: 1rem;
  right: -1.6rem;
  order: 2;
  border-left: 0;
  border-radius: 0 0.35rem 0.35rem 0;
}

.panel-toggle:hover {
  background: var(--sun-panel-color);
}

.sun-panel-reveal {
  position: absolute;
  top: 1rem;
  left: 0;
  z-index: 2;
  width: 1.5rem;
  height: 4rem;
  padding: 0;
  border: 0;
  border-radius: 0 0.5rem 0.5rem 0;
  color: #fff;
  background: var(--sun-panel-color);
  cursor: pointer;
  font: inherit;
  font-size: 1.15rem;
  opacity: 0;
  transition: opacity 0.2s ease;
}

.sun-panel-reveal:hover,
.sun-panel-reveal:focus-visible {
  opacity: 1;
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