<script setup>
import { ref, onMounted, shallowRef, markRaw, computed, watch, nextTick } from 'vue';
import MapBoard from './components/MapBoard.vue';
import shp from 'shpjs';
import L from 'leaflet';
import dayjs from 'dayjs';
import turfLength from '@turf/length';
import turfAlong from '@turf/along';
import lineSlice from '@turf/line-slice';
import { lineString, point } from '@turf/helpers';

const mapBoard = ref(null);
const fileInput = ref(null);
const shapeLayers = ref([]);
const tripUpdates = shallowRef(null);
const loadingShapes = ref(false);

// Playback State
const minTime = ref(0);
const maxTime = ref(0);
const currentTime = ref(0);
const isPlaying = ref(false);
const playbackSpeed = ref(10); // Multiplier? Or seconds per tick? Let's say speed multiplier
let animationValue = 0;
let lastFrameTime = 0;

onMounted(async () => {
  // Auto-load shapefiles from public directory
  loadingShapes.value = true;
  try {
    const response = await fetch('/shapefiles.zip');
    if (!response.ok) throw new Error("Failed to fetch shapefiles.zip");
    const buffer = await response.arrayBuffer();

    const geojson = await shp(buffer);
    console.log("Auto-loaded Shapefiles", geojson);

    if (Array.isArray(geojson)) {
      geojson.forEach(g => addShapeLayer(g, g.fileName));
    } else {
      addShapeLayer(geojson, 'Shapefile Layer');
    }
  } catch (err) {
    console.error("Error loading shapefiles:", err);
  } finally {
    loadingShapes.value = false;
  }

  requestAnimationFrame(animate);
});

const triggerUpload = () => {
  fileInput.value.click();
};

const handleFileUpload = async (e) => {
  const file = e.target.files[0];
  if (!file) return;

  if (file.name.endsWith('.json')) {
    const text = await file.text();
    try {
      const json = JSON.parse(text);
      tripUpdates.value = json;
      parseTimeRange(json);
    } catch (err) {
      console.error("Invalid JSON", err);
    }
  }
};

const parseTimeRange = (json) => {
  if (!json || !json.entity) return;

  let min = Infinity;
  let max = -Infinity;

  // Scan entities for time range
  json.entity.forEach(ent => {
    if (ent.tripUpdate && ent.tripUpdate.stopTimeUpdate) {
      ent.tripUpdate.stopTimeUpdate.forEach(stu => {
        const time = stu.arrival?.time || stu.departure?.time;
        if (time) {
          const t = parseInt(time);
          if (t < min) min = t;
          if (t > max) max = t;
        }
      });
    }
  });

  if (min !== Infinity) {
    minTime.value = min;
    maxTime.value = max;
    currentTime.value = min;
    console.log("Time Range:", new Date(min * 1000), "to", new Date(max * 1000));
  }
};

const togglePlay = () => {
  isPlaying.value = !isPlaying.value;
  if (isPlaying.value) {
    lastFrameTime = performance.now();
  }
};

const animate = (time) => {
  if (isPlaying.value && maxTime.value > minTime.value) {
    const delta = (time - lastFrameTime) / 1000; // seconds
    lastFrameTime = time;

    // Advance time: 30 seconds of data time per real second (speed factor can be adjusted)
    currentTime.value += delta * 30 * playbackSpeed.value;

    if (currentTime.value > maxTime.value) {
      currentTime.value = minTime.value; // Loop
    }
  } else {
    lastFrameTime = time;
  }
  requestAnimationFrame(animate);
};

const formattedTime = computed(() => {
  if (!currentTime.value) return "--:--";
  return dayjs.unix(currentTime.value).format('HH:mm:ss');
});

// Reuse existing shape logic
const addShapeLayer = (geojson, name) => {
  if (!mapBoard.value || !mapBoard.value.map) return;
  const map = mapBoard.value.map;

  const color = getRandomColor();
  const lowerName = name.toLowerCase();
  const isServiceArea = lowerName.includes('service') || lowerName.includes('area');
  const isHub = lowerName.includes('hub') || lowerName.includes('transit');

  const layer = L.geoJSON(geojson, {
    style: (feature) => {
      if (isServiceArea) return { color: color, weight: 1, fillOpacity: 0.1 };
      if (isHub) return { radius: 8, fillColor: '#0ff', color: '#000', weight: 1, fillOpacity: 0.8 };
      return { color: color, weight: 3, opacity: 0.8 };
    },
    pointToLayer: (feature, latlng) => {
      if (isHub) return L.circleMarker(latlng);
      return L.marker(latlng);
    },
    onEachFeature: (feature, layer) => {
      if (feature.properties) {
        let popupContent = `<div style="max-height: 200px; overflow-y: auto;"><b>${name}</b><br>`;
        for (const [key, val] of Object.entries(feature.properties)) {
          popupContent += `<b>${key}:</b> ${val}<br>`;
        }
        popupContent += "</div>";
        layer.bindPopup(popupContent);
      }
    }
  });

  layer.addTo(map);

  if (lowerName.includes('route') || lowerName.includes('service')) {
    map.fitBounds(layer.getBounds());
  }

  shapeLayers.value.push({
    id: name + '_' + Math.random().toString(36).substr(2, 9),
    name: name,
    visible: true,
    leafletLayer: markRaw(layer),
    color: (isHub ? '#0ff' : color)
  });
};

const toggleLayer = (layerItem) => {
  if (!mapBoard.value || !mapBoard.value.map) return;
  const map = mapBoard.value.map;
  if (layerItem.visible) {
    if (!map.hasLayer(layerItem.leafletLayer)) {
      layerItem.leafletLayer.addTo(map);
    }
  } else {
    map.removeLayer(layerItem.leafletLayer);
  }
}

const vehicleLayerGroup = ref(null); // LayerGroup for vehicles

// Initialize Vehicle Layer
onMounted(async () => {
  // ... existing loaded shapes ...
  if (mapBoard.value && mapBoard.value.map) {
    vehicleLayerGroup.value = L.layerGroup().addTo(mapBoard.value.map);
  }
});

// Optimization Caches
const routeFeatureMap = new Map();
const activeVehicleLayers = new Map();

// Watch shapes to rebuild cache
watch(shapeLayers, () => {
  rebuildRouteCache();
}, { deep: true });

const rebuildRouteCache = () => {
  routeFeatureMap.clear();
  console.log("Rebuilding Route Cache...");

  for (const layer of shapeLayers.value) {
    const geojson = layer.leafletLayer.toGeoJSON();
    let features = [];
    if (geojson.type === 'FeatureCollection') {
      features = geojson.features;
    } else if (geojson.type === 'Feature') {
      features = [geojson];
    }

    for (const f of features) {
      if (!f.properties) continue;
      // Scan for Route ID
      const props = f.properties;
      const possibleIds = [
        props.ROUTE_ID, props.route_id, props.RouteId,
        props.ROUTE, props.route, props.LINE_ID,
        // Numeric versions?
        props.ROUTE_ID?.toString(), props.route_id?.toString()
      ];

      // Allow string or number matches
      for (let id of possibleIds) {
        if (id !== undefined && id !== null) {
          routeFeatureMap.set(String(id), f);
        }
      }
      // Fuzzy fallback for "Route 1" vs "1"
      if (props.Shape_Name) {
        // Common GIS export format: "Route 10"
        // This is risky but often needed for messy data
      }
    }
  }
  console.log(`Route Cache built with ${routeFeatureMap.size} entries.`);
};

// Watch current time to update vehicles
watch(currentTime, () => {
  updateVehiclePositions();
});

const updateVehiclePositions = () => {
  if (!tripUpdates.value || !vehicleLayerGroup.value) return;

  // Set of trips visible in this frame (for GC)
  const visibleTrips = new Set();

  // Loop through trips
  for (const ent of tripUpdates.value.entity) {
    if (!ent.tripUpdate) continue;

    const tripId = ent.tripUpdate.trip.tripId;
    const routeId = ent.tripUpdate.trip.routeId;
    const routeIdStr = String(routeId);

    // 1. Fast Lookup (O(1))
    let lineFeature = routeFeatureMap.get(routeIdStr);

    if (!lineFeature) {
      // Optional: Try simple fuzzy fallback if strict match fails
      // (e.g. route "1" vs "Route 1")
      // For now, skip to save performance.
      continue;
    }

    // Ensure LineString
    lineFeature = getLineStringFeature(lineFeature);
    if (!lineFeature) continue;

    // 2. Calculate Position
    const currentPos = calculatePositionOnRoute(ent.tripUpdate, lineFeature);

    if (currentPos) {
      visibleTrips.add(tripId);

      // 3. Slice Route
      try {
        // Determine slice locally without creating L.geoJSON logic
        const startPt = point(lineFeature.geometry.coordinates[0]);
        const stopPt = point(currentPos);
        const sliced = lineSlice(startPt, stopPt, lineFeature);

        // Convert to Leaflet LatLngs (Flip coordinates: LonLat -> LatLon)
        const latLngs = sliced.geometry.coordinates.map(c => [c[1], c[0]]);

        // 4. Update or Create Layer (Reuse Phase)
        let layer = activeVehicleLayers.get(tripId);

        if (layer) {
          // Update existing (Fast DOM update)
          layer.setLatLngs(latLngs);
        } else {
          // Create new (Slow - happens once per trip)
          layer = L.polyline(latLngs, {
            color: '#FFD700',
            weight: 6,
            opacity: 1,
            lineCap: 'round'
          });

          layer.bindPopup(`<b>Route ${routeId}</b><br>Trip: ${tripId}`)
          layer.addTo(vehicleLayerGroup.value);
          activeVehicleLayers.set(tripId, layer);
        }
      } catch (e) {
        // console.warn(e);
      }
    }
  }

  // 5. Cleanup Inactive Layers (Garbage Collection)
  for (const [id, layer] of activeVehicleLayers) {
    if (!visibleTrips.has(id)) {
      vehicleLayerGroup.value.removeLayer(layer);
      activeVehicleLayers.delete(id);
    }
  }
};

const getLineStringFeature = (geojson) => {
  if (geojson.type === 'FeatureCollection') {
    const feature = geojson.features.find(f => f.geometry.type === 'LineString' || f.geometry.type === 'MultiLineString');
    if (feature) return feature;
  } else if (geojson.type === 'Feature') {
    return geojson;
  }
  return null;
}

const calculatePositionOnRoute = (tripUpdate, lineFeature) => {
  // 1. Get StopTimes
  const stopTimes = tripUpdate.stopTimeUpdate;
  if (!stopTimes || stopTimes.length < 2) return null;

  // 2. Find bounding stops for current time
  let prevStop = null;
  let nextStop = null;

  for (const st of stopTimes) {
    const time = parseInt(st.arrival?.time || st.departure?.time || 0);
    if (time <= currentTime.value) {
      prevStop = { ...st, time };
    } else {
      nextStop = { ...st, time };
      break; // Found the next stop
    }
  }

  if (!prevStop || !nextStop) return null; // Bus not currently running

  // 3. Interpolate progression (0 to 1) between stops
  const startTime = parseInt(stopTimes[0].arrival?.time || stopTimes[0].departure?.time);
  const endTime = parseInt(stopTimes[stopTimes.length - 1].arrival?.time || stopTimes[stopTimes.length - 1].departure?.time);

  if (currentTime.value < startTime || currentTime.value > endTime) return null;

  const totalProgress = (currentTime.value - startTime) / (endTime - startTime);

  return getPointAtFraction(lineFeature, totalProgress);
};

// Helper: Get point at fraction (0..1) along a LineString
const getPointAtFraction = (lineFeature, fraction) => {
  let line = lineFeature;
  try {
    if (line.geometry.type === 'MultiLineString') {
      line = lineString(line.geometry.coordinates[0]);
    }

    const len = turfLength(line);
    const dist = len * fraction;
    const pt = turfAlong(line, dist);
    return pt.geometry.coordinates; // [lon, lat]
  } catch (e) {
    return null;
  }
};

const getRandomColor = () => {
  const letters = '0123456789ABCDEF';
  let color = '#';
  for (let i = 0; i < 6; i++) {
    color += letters[Math.floor(Math.random() * 16)];
  }
  return color;
}

</script>

<template>
  <div class="app-container">
    <div class="top-bar">
      <div class="brand">
        <h1>Trip Viz</h1>
        <div class="controls" v-if="tripUpdates">
          <button class="btn-play" @click="togglePlay">{{ isPlaying ? '⏸' : '▶' }}</button>
          <div class="time-display">{{ formattedTime }}</div>
          <input type="range" class="timeline" :min="minTime" :max="maxTime" step="1" v-model.number="currentTime">
        </div>
        <div class="upload-section" v-else>
          <button class="btn-upload" @click="triggerUpload">📂 Load JSON</button>
          <input ref="fileInput" type="file" accept=".json" hidden @change="handleFileUpload">
        </div>
      </div>

      <div class="layers-panel">
        <span v-if="loadingShapes" class="loading-text">Loading Shapefiles...</span>
        <div v-else class="layer-list">
          <div v-for="layer in shapeLayers" :key="layer.id" class="layer-item">
            <label>
              <input type="checkbox" v-model="layer.visible" @change="toggleLayer(layer)">
              <span :style="{ backgroundColor: layer.color }" class="color-swatch"></span>
              {{ layer.name }}
            </label>
          </div>
        </div>
      </div>
    </div>

    <div class="map-view">
      <MapBoard ref="mapBoard" />
    </div>
  </div>
</template>

<style scoped>
.app-container {
  width: 100vw;
  height: 100vh;
  position: relative;
  overflow: hidden;
  background: #1e1e1e;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  color: #eee;
}

.top-bar {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 60px;
  background: rgba(30, 30, 30, 0.9);
  backdrop-filter: blur(10px);
  z-index: 1000;
  display: flex;
  align-items: center;
  padding: 0 90px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  gap: 20px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
}

.brand {
  display: flex;
  align-items: center;
  gap: 15px;
  min-width: 350px;
}

.brand h1 {
  margin: 0;
  font-size: 1.2rem;
  font-weight: 600;
  white-space: nowrap;
}

.controls {
  display: flex;
  align-items: center;
  gap: 10px;
  flex: 1;
}

.btn-play {
  background: #2ea043;
  border: none;
  color: white;
  width: 30px;
  height: 30px;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

.time-display {
  font-family: monospace;
  font-size: 1rem;
  width: 80px;
}

.timeline {
  flex: 1;
  cursor: pointer;
}

.btn-upload {
  background: #007acc;
  color: white;
  border: none;
  padding: 6px 12px;
  border-radius: 4px;
  cursor: pointer;
  font-weight: 500;
}

.btn-upload:hover {
  background: #0062a3;
}

.layers-panel {
  flex: 1;
  display: flex;
  align-items: center;
  overflow: hidden;
  border-left: 1px solid #444;
  padding-left: 15px;
}

.layer-list {
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 15px;
  overflow-x: auto;
  white-space: nowrap;
  padding-bottom: 5px;
}

.layer-list::-webkit-scrollbar {
  height: 4px;
}

.layer-list::-webkit-scrollbar-thumb {
  background: #555;
  border-radius: 2px;
}

.layer-item label {
  display: flex;
  align-items: center;
  gap: 6px;
  cursor: pointer;
  font-size: 0.9em;
  background: rgba(255, 255, 255, 0.05);
  padding: 4px 10px;
  border-radius: 4px;
  transition: background 0.2s;
  user-select: none;
}

.layer-item label:hover {
  background: rgba(255, 255, 255, 0.15);
}

.color-swatch {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  display: inline-block;
  border: 1px solid rgba(255, 255, 255, 0.5);
}

.map-view {
  width: 100%;
  height: 100%;
}
</style>
