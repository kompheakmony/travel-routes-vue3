<template>
  <div class="route-animation">
    <div ref="mapContainer" class="map-container"></div>

    <div v-if="error" class="error-message">{{ error }}</div>

    <div v-if="routeFeatures.length" class="controls">
      <!-- Route Selector -->
      <div class="control-group">
        <label for="route-select" class="sr-only">Select Route</label>
        <select id="route-select" v-model="selectedRouteIndex" title="Select a route">
          <option v-for="(route, index) in routeFeatures" :value="index" :key="index">
            {{ route.properties?.name || `Route ${index + 1}` }}
          </option>
        </select>
      </div>

      <!-- Animation Controls -->
      <div class="control-group action-buttons">
        <button @click="toggleAnimation" :disabled="!selectedRouteCoordinates.length"
          :title="isAnimating ? 'Pause Animation' : 'Start Animation'">
          <svg v-if="isAnimating" viewBox="0 0 24 24" fill="currentColor" width="24" height="24">
            <path d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"></path>
          </svg>
          <svg v-else viewBox="0 0 24 24" fill="currentColor" width="24" height="24">
            <path d="M8 5v14l11-7z"></path>
          </svg>
        </button>

        <button @click="resetAnimation" :disabled="!selectedRouteCoordinates.length" title="Reset Animation">
          <svg viewBox="0 0 24 24" fill="currentColor" width="24" height="24">
            <path d="M12 5V1L7 6l5 5V7c3.31 0 6 2.69 6 6s-2.69 6-6 6-6-2.69-6-6H4c0 4.42 3.58 8 8 8s8-3.58 8-8-3.58-8-8-8z"></path>
          </svg>
        </button>
      </div>

      <!-- REC Indicator -->
      <div class="rec-indicator" v-if="isRecording">
        <span class="rec-dot"></span>REC
      </div>

      <!-- Speed Control -->
      <div class="control-group speed-control">
        <label for="speed-range" class="sr-only">Animation Speed</label>
        <input id="speed-range" type="range" v-model.number="animationSpeed" min="10" max="300" step="5" title="Adjust animation speed">
        <span class="speed-display">{{ animationSpeed }} km/h</span>
      </div>
      
      <!-- Download Button -->
      <a v-if="downloadUrl" :href="downloadUrl" download="map-animation.webm" class="btn-download" title="Download Video">
        <svg viewBox="0 0 24 24" fill="currentColor" width="24" height="24">
          <path d="M19 9h-4V3H9v6H5l7 7 7-7zM5 18v2h14v-2H5z"></path>
        </svg>
        <span>Download</span>
      </a>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, computed, watch } from 'vue';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import html2canvas from 'html2canvas';

const props = defineProps({
  geojsonUrl: { type: String, default: '/public/data/map.geojson' },
  mapCenter: { type: Array, default: () => [11.556, 104.928] },
  mapZoom: { type: Number, default: 16 }
});

const mapContainer = ref(null);
const map = ref(null);
const routeData = ref(null);
const routePolylineLayers = ref({});
const movingIconMarker = ref(null);
const progressPolyline = ref(null);

const canvasRecorder = ref(null);
const canvasContext = ref(null);
const recordingCanvas = ref(null);
const recordedChunks = ref([]);
const mediaRecorder = ref(null);
const downloadUrl = ref(null);
const isRecording = ref(false);
let captureInterval = null;

const isAnimating = ref(false);
const selectedRouteIndex = ref(0);
const animationSpeed = ref(145);
const animationFrameId = ref(null);
const progress = ref(0);
const error = ref(null);
const totalRouteDistance = ref(0);

const routeFeatures = computed(() => (routeData.value?.features || []).filter(f => f.geometry?.type === 'LineString'));
const selectedRoute = computed(() => routeFeatures.value[selectedRouteIndex.value] || null);
const selectedRouteCoordinates = computed(() => selectedRoute.value?.geometry.coordinates.map(coord => L.latLng(coord[1], coord[0])) || []);

onMounted(async () => {
  try {
    const response = await fetch(props.geojsonUrl);
    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
    routeData.value = await response.json();
    initializeMap();
    drawAllFeatures();
    updateSelectedRouteDisplay(null, selectedRouteIndex.value);
    
    setTimeout(() => map.value.invalidateSize(), 300);
  } catch (err) {
    console.error('Error during component setup:', err);
    error.value = `Could not load map data. Please check the console.`;
  }
});

onBeforeUnmount(() => {
  stopAnimation();
  if (map.value) map.value.remove();
});

watch(selectedRouteIndex, (newIndex, oldIndex) => {
  resetAnimation();
  updateSelectedRouteDisplay(oldIndex, newIndex);
});

watch(isAnimating, (isNowAnimating) => {
  if (isNowAnimating) startAnimation();
  else stopAnimation();
});

function initializeMap() {
  if (mapContainer.value) {
    map.value = L.map(mapContainer.value).setView(props.mapCenter, props.mapZoom);
    L.tileLayer('https://{s}.basemaps.cartocdn.com/rastertiles/voyager/{z}/{x}/{y}{r}.png', {
      attribution: '© <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors © <a href="https://carto.com/attributions">CARTO</a>',
      maxZoom: 30,
    }).addTo(map.value);
  }
}

function startRecording() {
  const target = mapContainer.value;
  if (!target || isRecording.value) return;

  recordingCanvas.value = document.createElement('canvas');
  recordingCanvas.value.width = 1920;
  recordingCanvas.value.height = 1080;
  canvasContext.value = recordingCanvas.value.getContext('2d');

  const stream = recordingCanvas.value.captureStream(30);
  recordedChunks.value = [];
  mediaRecorder.value = new MediaRecorder(stream, { mimeType: 'video/webm' });

  mediaRecorder.value.ondataavailable = (e) => {
    if (e.data.size > 0) recordedChunks.value.push(e.data);
  };

  mediaRecorder.value.onstop = () => {
    const blob = new Blob(recordedChunks.value, { type: 'video/webm' });
    downloadUrl.value = URL.createObjectURL(blob);
  };

  mediaRecorder.value.start();
  isRecording.value = true;

  captureInterval = setInterval(() => {
    html2canvas(target, { backgroundColor: null, useCORS: true })
      .then((canvas) => {
        canvasContext.value.clearRect(0, 0, 1920, 1080);
        canvasContext.value.drawImage(canvas, 0, 0, 1920, 1080);
    });
  }, 33);
}

function stopRecording() {
  if (mediaRecorder.value && isRecording.value) {
    clearInterval(captureInterval);
    mediaRecorder.value.stop();
    isRecording.value = false;
  }
}

function drawAllFeatures() {
  if (!map.value || !routeData.value) return;

  routeFeatures.value.forEach((feature, index) => {
    const latLngs = feature.geometry.coordinates.map(coord => [coord[1], coord[0]]);
    routePolylineLayers.value[index] = L.polyline(latLngs, {
      color: '#888', weight: 5, opacity: 0.7,
    }).addTo(map.value);
  });

  const pointFeatures = (routeData.value.features || []).filter(f => f.geometry?.type === 'Point');
  pointFeatures.forEach(feature => {
    const coords = [feature.geometry.coordinates[1], feature.geometry.coordinates[0]];
    const marker = L.marker(coords, {
      icon: L.divIcon({
        className: 'custom-point-marker',
        html: `<div style="background-color:${feature.properties?.['marker-color'] || '#EA4335'};"></div>`,
        iconSize: [16, 16],
        iconAnchor: [8, 8], // Centered anchor for the marker
      })
    }).addTo(map.value);

    if (feature.properties?.name) {
      marker.bindPopup(`<strong>${feature.properties.name}</strong>`);
    }
  });
}

function updateSelectedRouteDisplay(oldIndex, newIndex) {
  if (oldIndex !== null && routePolylineLayers.value[oldIndex]) {
    routePolylineLayers.value[oldIndex].setStyle({ color: '#888', weight: 5, dashArray: null });
  }

  const newPolyline = routePolylineLayers.value[newIndex];
  if (newPolyline) {
    newPolyline.setStyle({ color: '#555', weight: 7, dashArray: '10, 10' });
    newPolyline.bringToFront();
    map.value.fitBounds(newPolyline.getBounds().pad(0.1));
  }

  if (!progressPolyline.value) {
    progressPolyline.value = L.polyline([], {
      color: selectedRoute.value?.properties?.stroke || '#28a745',
      weight: 7,
      opacity: 1,
    }).addTo(map.value);
  } else {
    progressPolyline.value.setStyle({ color: selectedRoute.value?.properties?.stroke || '#4285F4' });
  }
  progressPolyline.value.setLatLngs([]);
  progressPolyline.value.bringToFront();
  totalRouteDistance.value = calculateTotalDistance(selectedRouteCoordinates.value);
}

function toggleAnimation() { isAnimating.value = !isAnimating.value; }

function startAnimation() {
  if (selectedRouteCoordinates.value.length < 2) {
    isAnimating.value = false;
    return;
  }
  if (!movingIconMarker.value) createMovingIcon();
  downloadUrl.value = null; // Clear previous download link
  startRecording(); 
  animate();
}

function stopAnimation() {
  if (animationFrameId.value) {
    cancelAnimationFrame(animationFrameId.value);
    animationFrameId.value = null;
    startTime = 0;
  }
}

function resetAnimation() {
  stopRecording();
  isAnimating.value = false;
  progress.value = 0;
  startTime = null;
  downloadUrl.value = null;

  if (movingIconMarker.value && selectedRouteCoordinates.value.length > 0) {
    const startLatLng = selectedRouteCoordinates.value[0];
    movingIconMarker.value.setLatLng(startLatLng);
    map.value.panTo(startLatLng);
  }

  if (progressPolyline.value) {
    progressPolyline.value.setLatLngs([]);
  }
}

let startTime = null;

const kmPerHourToMetersPerMs = (kmh) => kmh * 1000 / 3600;

function animate(timestamp) {
  if (!isAnimating.value) return;

  if (!startTime) startTime = timestamp;

  const elapsed = timestamp - startTime;
  const totalDist = calculateTotalDistance(selectedRouteCoordinates.value);
  const speed = animationSpeed.value;
  const speedMetersPerMs = kmPerHourToMetersPerMs(speed);

  // Using a multiplier to make the animation visually pleasing, not physically accurate.
  const effectiveSpeed = speedMetersPerMs * 0.2;
  const estimatedDuration = totalDist / effectiveSpeed;

  progress.value = Math.min(elapsed / estimatedDuration, 1);

  updateMarkerPosition();

  if (progress.value >= 1) {
    isAnimating.value = false;
    startTime = null;
    stopRecording();
    return;
  }

  animationFrameId.value = requestAnimationFrame(animate);
}

function updateMarkerPosition() {
  const coords = selectedRouteCoordinates.value;
  const totalDist = calculateTotalDistance(coords);
  const currentDist = totalDist * progress.value;
  const { latLng } = getLatLngAndRotationAtDistance(coords, currentDist);

  if (latLng && movingIconMarker.value) {
    movingIconMarker.value.setLatLng(latLng);

    // FIX: Use animationSpeed.value to get the current speed
    const speed = animationSpeed.value;
    const minZoom = 12;
    const maxZoom = 17;
    const zoom = maxZoom - ((speed - 10) / (300 - 10)) * (maxZoom - minZoom);
    const clampedZoom = Math.min(Math.max(zoom, minZoom), maxZoom);

    map.value.setView(latLng, clampedZoom, { animate: false, pan: { duration: 0.1 } });
  }

  if (progressPolyline.value) {
    const progressLatLngs = getCoordinatesForProgress(coords, currentDist);
    progressPolyline.value.setLatLngs(progressLatLngs);
  }
}

function createMovingIcon() {
  const startLatLng = selectedRouteCoordinates.value[0];
  const movingDot = L.divIcon({
    className: 'moving-icon-dot',
    html: `<div class="dot pulse"></div>`,
    iconSize: [20, 20],
    iconAnchor: [10, 10],
  });

  movingIconMarker.value = L.marker(startLatLng, {
    icon: movingDot,
    zIndexOffset: 1000,
  }).addTo(map.value);

  resetAnimation();
}

function calculateTotalDistance(latLngs) {
  let total = 0;
  for (let i = 0; i < latLngs.length - 1; i++) {
    total += latLngs[i].distanceTo(latLngs[i + 1]);
  }
  return total;
}

function getLatLngAndRotationAtDistance(latLngs, distance) {
  if (distance <= 0) return { latLng: latLngs[0], rotation: 0 };
  let traveled = 0;
  for (let i = 0; i < latLngs.length - 1; i++) {
    const start = latLngs[i];
    const end = latLngs[i + 1];
    const segmentDistance = start.distanceTo(end);
    if (traveled + segmentDistance >= distance) {
      const ratio = (distance - traveled) / segmentDistance;
      const lat = start.lat + (end.lat - start.lat) * ratio;
      const lng = start.lng + (end.lng - start.lng) * ratio;
      return { latLng: L.latLng(lat, lng), rotation: 0 };
    }
    traveled += segmentDistance;
  }
  return { latLng: latLngs[latLngs.length - 1], rotation: 0 };
}

function getCoordinatesForProgress(latLngs, distance) {
  const progressCoords = [];
  if (distance <= 0 || latLngs.length < 2) return latLngs.length ? [latLngs[0]] : [];
  let traveled = 0;
  for (let i = 0; i < latLngs.length - 1; i++) {
    const start = latLngs[i];
    const end = latLngs[i + 1];
    const segmentDistance = start.distanceTo(end);
    if (traveled + segmentDistance >= distance) {
      const ratio = (distance - traveled) / segmentDistance;
      const lat = start.lat + (end.lat - start.lat) * ratio;
      const lng = start.lng + (end.lng - start.lng) * ratio;
      progressCoords.push(...latLngs.slice(0, i + 1));
      progressCoords.push(L.latLng(lat, lng));
      return progressCoords;
    }
    traveled += segmentDistance;
  }
  return [...latLngs];
}
</script>

<style scoped>
/* General Layout & Theme */
.route-animation {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif, "Apple Color Emoji", "Segoe UI Emoji", "Segoe UI Symbol";
}

.map-container {
  width: 100%;
  height: 100%;
  background-color: #aad3df; /* A pleasant placeholder color */
}

/* Visually hidden class for accessibility */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border-width: 0;
}

/* Controls Bar */
.controls {
  position: absolute;
  bottom: 25px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 1000;
  
  display: flex;
  align-items: center;
  gap: 12px;
  
  background: rgba(25, 28, 34, 0.8);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  padding: 8px 16px;
  border-radius: 50px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
  color: #f0f0f0;
}

.control-group {
  display: flex;
  align-items: center;
  gap: 8px;
}

/* Action Buttons (Play, Pause, Reset) */
.action-buttons button {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  padding: 0;
  background-color: transparent;
  border: none;
  border-radius: 50%;
  color: #f0f0f0;
  cursor: pointer;
  transition: background-color 0.2s ease-in-out;
}

.action-buttons button:hover:not(:disabled) {
  background-color: rgba(255, 255, 255, 0.15);
}

.action-buttons button:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.action-buttons button svg {
  width: 24px;
  height: 24px;
}

/* Route Selector Dropdown */
.controls select {
  background-color: rgba(255, 255, 255, 0.1);
  color: #f0f0f0;
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 20px;
  padding: 8px 16px;
  -webkit-appearance: none;
  -moz-appearance: none;
  appearance: none;
  background-image: url("data:image/svg+xml;charset=UTF-8,%3csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%23f0f0f0' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3e%3cpolyline points='6 9 12 15 18 9'%3e%3c/polyline%3e%3c/svg%3e");
  background-repeat: no-repeat;
  background-position: right 12px center;
  background-size: 1em;
  padding-right: 32px; /* Make space for arrow */
}

.controls select:focus {
  outline: none;
  border-color: #4285F4;
}

/* Speed Slider */
.speed-control {
  padding: 0 10px;
}

.speed-control input[type="range"] {
  -webkit-appearance: none;
  appearance: none;
  width: 150px;
  height: 4px;
  background: rgba(255, 255, 255, 0.3);
  border-radius: 2px;
  outline: none;
  cursor: pointer;
}

.speed-control input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 16px;
  height: 16px;
  background: #f0f0f0;
  border-radius: 50%;
  border: 2px solid #333;
  transition: background-color 0.2s ease;
}
.speed-control input[type="range"]::-moz-range-thumb {
  width: 14px;
  height: 14px;
  background: #f0f0f0;
  border-radius: 50%;
  border: 2px solid #333;
  transition: background-color 0.2s ease;
}

.speed-display {
  font-family: 'SF Mono', 'Menlo', 'Monaco', 'Consolas', 'Liberation Mono', 'Courier New', monospace;
  font-size: 0.9em;
  min-width: 80px;
  text-align: right;
  background-color: rgba(0,0,0,0.2);
  padding: 4px 8px;
  border-radius: 4px;
}

/* Recording Indicator */
.rec-indicator {
  display: flex;
  align-items: center;
  gap: 6px;
  color: #ff5252;
  font-weight: 600;
  font-size: 0.8em;
  letter-spacing: 0.5px;
  padding: 0 5px;
}
.rec-indicator .rec-dot {
  width: 8px;
  height: 8px;
  background-color: #ff5252;
  border-radius: 50%;
  animation: rec-pulse 1.5s infinite ease-in-out;
}
@keyframes rec-pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.5; transform: scale(0.8); }
}

/* Download Button */
.btn-download {
  display: flex;
  align-items: center;
  gap: 8px;
  background-color: #4285F4;
  color: white;
  text-decoration: none;
  padding: 8px 16px;
  border-radius: 20px;
  font-weight: 500;
  font-size: 0.9em;
  transition: background-color 0.2s ease;
}
.btn-download:hover {
  background-color: #5a95f5;
}
.btn-download svg {
  width: 20px;
  height: 20px;
}

/* Error Message */
.error-message {
  position: absolute;
  top: 15px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 1000;
  background-color: rgba(217, 48, 37, 0.85);
  color: white;
  padding: 12px 24px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
  font-weight: 500;
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

/* --- Map Element Styles --- */

/* Use :deep() to style Leaflet's injected HTML elements from a scoped style block */
:deep(.custom-point-marker div) {
  width: 12px !important;
  height: 12px !important;
  border-radius: 50%;
  border: 2px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.5);
}

:deep(.moving-icon-dot .dot) {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background-color: #4285F4;
  border: 2px solid #fff;
  box-shadow: 0 0 10px rgba(66, 133, 244, 0.7);
}

:deep(.pulse) {
  /* This creates an outer ring that expands and fades, a common map marker effect */
  box-shadow: 0 0 0 0 rgba(66, 133, 244, 0.7);
  animation: ripple-pulse 1.75s infinite;
}

@keyframes ripple-pulse {
  0% {
    transform: scale(0.95);
    box-shadow: 0 0 0 0 rgba(66, 133, 244, 0.7);
  }
  70% {
    transform: scale(1);
    box-shadow: 0 0 0 15px rgba(66, 133, 244, 0);
  }
  100% {
    transform: scale(0.95);
    box-shadow: 0 0 0 0 rgba(66, 133, 244, 0);
  }
}
</style>