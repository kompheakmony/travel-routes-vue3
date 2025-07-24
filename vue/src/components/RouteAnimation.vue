<template>
  <div class="route-animation">
    <div ref="mapContainer" class="map-container"></div>
    <div v-if="routeFeatures.length" class="controls">
      <select v-model="selectedRouteIndex" title="Select a route">
        <option v-for="(route, index) in routeFeatures" :value="index" :key="index">
          {{ route.properties?.name || `Route ${index + 1}` }}
        </option>
      </select>
      <button @click="toggleAnimation" :disabled="!selectedRouteCoordinates.length" :title="isAnimating ? 'Pause Animation' : 'Start Animation'">
        <span v-if="isAnimating">Pause</span>
        <span v-else>Start</span>
      </button>
      <button @click="resetAnimation" :disabled="!selectedRouteCoordinates.length" title="Reset Animation">
        Reset
      </button>
      <input type="range" v-model.number="animationSpeed" min="10" max="300" step="5" title="Adjust animation speed">
      <span class="speed-display">{{ animationSpeed }} km/h</span>
    </div>
    <div v-if="error" class="error-message">{{ error }}</div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, computed, watch } from 'vue';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';

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

const isAnimating = ref(false);
const selectedRouteIndex = ref(0);
const animationSpeed = ref(90);
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
  } catch (err) {
    console.error('Error during component setup:', err);
    error.value = `Could not load map data.`;
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
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '© OpenStreetMap contributors', maxZoom: 19,
    }).addTo(map.value);
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
      })
    }).addTo(map.value);

    if (feature.properties?.name) {
      marker.bindPopup(`<strong>${feature.properties.name}</strong>`);
    }
  });
}

function updateSelectedRouteDisplay(oldIndex, newIndex) {
  if (oldIndex !== null && routePolylineLayers.value[oldIndex]) {
    routePolylineLayers.value[oldIndex].setStyle({ color: '#888', weight: 5 });
  }

  const newPolyline = routePolylineLayers.value[newIndex];
  if (newPolyline) {
    newPolyline.setStyle({ color: '#888', weight: 7, dashArray: '10, 10' });
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
  isAnimating.value = false;
  progress.value = 0;
  startTime = null;

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

  const effectiveSpeed = speedMetersPerMs * 0.2;
  const estimatedDuration = totalDist / effectiveSpeed;

  progress.value = Math.min(elapsed / estimatedDuration, 1);

  updateMarkerPosition();

  if (progress.value >= 1) {
    isAnimating.value = false;
    startTime = null;
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

    const speed = animationSpeed.value;
    const minZoom = 6;
    const maxZoom = 16;
    const zoom = Math.round(maxZoom - (speed / 300) * (maxZoom - minZoom));
    const clampedZoom = Math.min(Math.max(zoom, minZoom), maxZoom);

    map.value.setView(latLng, clampedZoom, { animate: true, duration: 0.5 });
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
  if (distance <= 0 || latLngs.length < 2) return [latLngs[0]];
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
.route-animation {
  position: relative;
  height: 100%;
  width: 100%;
}
.map-container {
  height: 600px;
  width: 100%;
  z-index: 1;
  background-color: #f0f0f0;
}
.controls {
  position: absolute;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 1000;
  background: white;
  padding: 10px 15px;
  border-radius: 25px;
  display: flex;
  gap: 12px;
  align-items: center;
  box-shadow: 0 2px 15px rgba(0,0,0,0.2);
}
.speed-display {
  font-family: monospace;
  min-width: 75px;
  text-align: center;
}
.error-message {
  position: absolute;
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 1000;
  background-color: #f8d7da;
  color: #721c24;
  padding: 10px 20px;
  border: 1px solid #f5c6cb;
  border-radius: 5px;
}
.moving-icon-dot .dot {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background-color: #1267ff;
  border: 2px solid #fff;
  box-shadow: 0 0 8px rgba(18, 103, 255, 0.5);
}
.pulse {
  animation: pulse 1s infinite;
}
@keyframes pulse {
  0% { transform: scale(1); opacity: 1; }
  50% { transform: scale(1.5); opacity: 0.5; }
  100% { transform: scale(1); opacity: 1; }
}
</style>
