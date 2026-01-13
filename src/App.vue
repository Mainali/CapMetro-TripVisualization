<script setup>
import { ref, onMounted } from 'vue';
import MapBoard from './components/MapBoard.vue';
import shp from 'shpjs';
import L from 'leaflet';

const mapBoard = ref(null);
const dragging = ref(false);
const shapeLayers = ref([]); // Array of { id, name, visible, leafletLayer, color }
const tripUpdates = ref(null);
const loadingShapes = ref(false);

onMounted(async () => {
  // Auto-load shapefiles from public directory
  loadingShapes.value = true;
  try {
    const response = await fetch('/shapefiles.zip');
    if (!response.ok) throw new Error("Failed to fetch shapefiles.zip");
    const buffer = await response.arrayBuffer();

    // shpjs returns an array of features if multiple shapefiles in zip
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
});

const handleDrop = async (e) => {
  e.preventDefault();
  dragging.value = false;

  const files = e.dataTransfer.files;
  for (const file of files) {
    if (file.name.endsWith('.json')) {
      const text = await file.text();
      try {
        const json = JSON.parse(text);
        tripUpdates.value = json;
        console.log("Loaded trip updates", json);
      } catch (err) {
        console.error("Invalid JSON", err);
      }
    } else if (file.name.endsWith('.zip')) {
      // Allow dropping other shapefiles
      const buffer = await file.arrayBuffer();
      const geojson = await shp(buffer);
      if (Array.isArray(geojson)) {
        geojson.forEach(g => addShapeLayer(g, g.fileName));
      } else {
        addShapeLayer(geojson, file.name);
      }
    }
  }
};

const addShapeLayer = (geojson, name) => {
  if (!mapBoard.value || !mapBoard.value.map) return;
  const map = mapBoard.value.map;

  const color = getRandomColor();
  // Heuristics for style
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

  // Zoom if it's a significant layer (Route/Service area)
  if (lowerName.includes('route') || isServiceArea) {
    map.fitBounds(layer.getBounds());
  }

  shapeLayers.value.push({
    id: name + '_' + Math.random().toString(36).substr(2, 9),
    name: name,
    visible: true,
    leafletLayer: layer,
    color: (isHub ? '#0ff' : color)
  });
};

const toggleLayer = (layerItem) => {
  if (!mapBoard.value || !mapBoard.value.map) return;
  const map = mapBoard.value.map;

  if (layerItem.visible) {
    layerItem.leafletLayer.addTo(map);
  } else {
    map.removeLayer(layerItem.leafletLayer);
  }
}

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
  <div class="app-container" @dragover.prevent="dragging = true" @dragleave.prevent="dragging = false"
    @drop="handleDrop">

    <div class="top-bar">
      <div class="brand">
        <h1>Trip Viz</h1>
        <div class="status-badge" v-if="tripUpdates">
          {{ tripUpdates.entity.length }} Updates
        </div>
        <div class="status-badge empty" v-else>
          Drag 'tripupdates.json'
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
      <div v-if="dragging" class="drop-overlay">
        <h2>Drop files to load</h2>
      </div>
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
}

.top-bar {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 60px;
  background: rgba(30, 30, 30, 0.85);
  backdrop-filter: blur(5px);
  z-index: 1000;
  display: flex;
  align-items: center;
  padding: 0 20px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  gap: 20px;
  color: #eee;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
}

.brand {
  display: flex;
  align-items: center;
  gap: 15px;
  min-width: 200px;
}

.brand h1 {
  margin: 0;
  font-size: 1.2rem;
  font-weight: 600;
  letter-spacing: 0.5px;
}

.status-badge {
  background: #2ea043;
  color: white;
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 0.8rem;
  font-weight: 500;
}

.status-badge.empty {
  background: #444;
  color: #ccc;
  border: 1px dashed #777;
}

.layers-panel {
  flex: 1;
  display: flex;
  align-items: center;
  overflow: hidden;
}

.layer-list {
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 15px;
  overflow-x: auto;
  white-space: nowrap;
  padding-bottom: 5px;
  /* space for scrollbar if needed */
}

/* Scrollbar styling for horizontal list */
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

.drop-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  pointer-events: none;
}
</style>
<style>
body {
  margin: 0;
  padding: 0;
  font-family: sans-serif;
}
</style>
