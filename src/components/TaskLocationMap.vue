<template>
  <div
    ref="mapElement"
    class="task-location-map"
    aria-label="Escolha a localização no mapa"
  />
</template>

<script setup>
import { nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

import markerIcon from 'leaflet/dist/images/marker-icon.png'
import markerIcon2x from 'leaflet/dist/images/marker-icon-2x.png'
import markerShadow from 'leaflet/dist/images/marker-shadow.png'

L.Icon.Default.mergeOptions({
  iconRetinaUrl: markerIcon2x,
  iconUrl: markerIcon,
  shadowUrl: markerShadow,
})

const props = defineProps({
  location: {
    type: Object,
    default: null,
  },
})

const emit = defineEmits(['select'])

const mapElement = ref(null)

let map
let marker

function setMarker(latitude, longitude) {
  const point = [latitude, longitude]

  if (marker) {
    marker.setLatLng(point)
  } else {
    marker = L.marker(point).addTo(map)
  }

  map.setView(point, 17)
}

function handleMapClick(event) {
  const { lat, lng } = event.latlng

  setMarker(lat, lng)

  emit('select', {
    latitude: lat,
    longitude: lng,
    accuracy: null,
    timestamp: Date.now(),
    label: null,
  })
}

function renderLocation() {
  if (!map || !props.location) return

  const { latitude, longitude } = props.location

  if (latitude == null || longitude == null) return

  setMarker(latitude, longitude)

  if (props.location.label) {
    marker
      .bindPopup(props.location.label)
      .openPopup()
  }

  nextTick(() => {
    map.invalidateSize()
  })
}

onMounted(() => {
  map = L.map(mapElement.value).setView([-26.9, -49.0], 12)

  L.tileLayer(
    'https://tile.openstreetmap.org/{z}/{x}/{y}.png',
    {
      attribution:
        '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
    },
  ).addTo(map)

  map.on('click', handleMapClick)

  renderLocation()

  nextTick(() => {
    map.invalidateSize()
  })
})

watch(
  () => props.location,
  () => {
    renderLocation()
  },
  { deep: true },
)

onBeforeUnmount(() => {
  if (map) {
    map.remove()
  }
})
</script>

<style scoped>
.task-location-map {
  width: 100%;
  height: 300px;
  margin-top: 12px;
  border-radius: 8px;
  overflow: hidden;
}
</style>