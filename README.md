# Trip Analysis GIS Visualization

A high-performance geospatial web application for visualizing public transit trip updates in real-time. Built with Vue 3, Leaflet, and Turf.js.

<video controls src="./public/screenRecord.mp4" width="100%" autoplay loop muted></video>

---

## 🚀 Features

- **Real-Time Visualization**: Visualizes `tripupdates.json` data on an interactive map.
- **Dynamic Route Interpolation**: Uses **Turf.js** to slice route geometries and render the exact progress of buses along their paths based on timestamps.
- **Shapefile Integration**: Automatically loads and parses zipped Shapefiles (Routes, Service Areas, Transit Hubs) using `shpjs`.
- **Playback Controls**:
  - Time Slider for scrubbing through history.
  - Play/Pause animation with adjustable speed.
  - Real-time clock display.
- **Performance Optimized**:
  - **Route Caching**: O(1) lookup for route geometries using a pre-built feature cache.
  - **Layer Reuse**: Efficiently updates Leaflet layers to maintain 60FPS during playback.
- **Data Import**: Supports uploading custom JSON trip update files.

## 🛠️ Technology Stack

- **Frontend Framework**: [Vue.js 3](https://vuejs.org/) (Vite)
- **Mapping Engine**: [Leaflet](https://leafletjs.com/)
- **Geospatial Analysis**: [Turf.js](https://turfjs.org/) (Mileage, Slicing, Interpolation)
- **Data Parsing**: 
  - `shpjs` (Shapefiles)
  - `dayjs` (Time formatting)

## 📦 Data Sources

- **GTFS / Trip Updates**: [CapMetro Developer Tools](https://www.capmetro.org/developertools)
- **Shapefiles**: Derived from CapMetro GIS data (Routes, Service Areas, Hubs).

## ⚡ Setup & Running

1. **Install Dependencies**
   ```bash
   npm install
   ```

2. **Run Development Server**
   ```bash
   npm run dev
   ```

3. **Open Browser**
   Navigate to `http://localhost:5173`.

## 📂 Project Structure

- `src/App.vue`: Main application logic (State, Playback Loop, Rendering).
- `src/components/MapBoard.vue`: Leaflet map initialization.
- `public/shapefiles.zip`: Default geospatial data loaded on startup.
- `public/screenRecord.mp4`: Demo recording.
