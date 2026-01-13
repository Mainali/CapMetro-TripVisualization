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

    <div class="sidebar">
      <h1>Trip Viz</h1>
      <div v-if="tripUpdates">
        <p>Updates: {{ tripUpdates.entity.length }}</p>
      </div>
      <div v-else>
        <p>Drag 'tripupdates.json' here</p>
      </div>

      <div class="layers-panel">
        <h3>Layers</h3>
        <p v-if="loadingShapes">Loading Shapefiles...</p>
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
  display: flex;
  height: 100vh;
  width: 100vw;
  overflow: hidden;
  background: #1e1e1e;
  color: #eee;
}

.sidebar {
  width: 300px;
  background: #252526;
  padding: 20px;
  border-right: 1px solid #333;
  z-index: 1000;
}

.layers-panel {
  margin-top: 20px;
  border-top: 1px solid #444;
  padding-top: 10px;
}

.layer-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  overflow-y: auto;
  max-height: 50vh;
}

.layer-item label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  font-size: 0.9em;
}

.color-swatch {
  width: 12px;
  height: 12px;
  border-radius: 2px;
  border: 1px solid #fff;
  display: inline-block;
}

.map-view {
  flex: 1;
  position: relative;
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
