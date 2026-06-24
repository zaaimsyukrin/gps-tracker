<script setup>
import { ref, onMounted } from 'vue'
import { LMap, LTileLayer, LPolyline, LMarker, LCircleMarker } from '@vue-leaflet/vue-leaflet'
import 'leaflet/dist/leaflet.css'
import { toPng } from 'html-to-image'
import { Filesystem, Directory } from '@capacitor/filesystem'
import { Media } from '@capacitor-community/media'


const zoom = ref(16) // How zoomed in the map is
const currentPosition = ref([3.1390, 101.6869]) // KL 
const startMarker = ref(null)
const endMarker = ref(null)
const pathPoints = ref([]) // Stores gps coordinates every second
const markedPoints = ref([]) // Stores marker coordinates
const isTracking = ref(false) // Bool for disabling buttons when app is tracking
let watchId = null
let markerId = 0 // Id for each marker placed by "+" button

/**
 * Called when "start" button is pressed
 * - Drop green marker at the beginning
 * - Every current position gets added to the pathPoints array
 */
function startTracking() {
  isTracking.value = true
  let isFirstPoint = true  // Flag for first gps reading
  let isWarmedUp = false  // Flag for warm-up period

  // Wait 4 seconds before recording (idk why but GPS gets crazy in the first 4 secs)
  setTimeout(() => {
    isWarmedUp = true
  }, 4000)

  watchId = navigator.geolocation.watchPosition(
    (position) => {
      if (!isWarmedUp) return  // Wait 4 secs for GPS to stabilize
      if (position.coords.accuracy > 50) 
        return // Any outliers (>50 meters) gets ignored

      // Get current position, and append it into pathPoints array
      const { latitude, longitude } = position.coords
      currentPosition.value = [latitude, longitude]
      pathPoints.value = [...pathPoints.value, [latitude, longitude]]

      // Only set start marker on first real gps reading
      if (isFirstPoint) {
        startMarker.value = [latitude, longitude]
        isFirstPoint = false
      }
    },
    (error) => {
      alert('GPS error: ' + error.message)
    },
    { enableHighAccuracy: true }
  )
}

/**
 * Called when "stop" button is pressed.
 * - save snapshot of map to album "GPS Tracker" in gallery
 */
async function stopTracking() {
  endMarker.value = currentPosition.value
  isTracking.value = false
  navigator.geolocation.clearWatch(watchId) // Stop tracking current loc

  // -- Saving snapshot of map to gallery --
  const mapElement = document.getElementById('map')
  const dataUrl = await toPng(mapElement, { cacheBust: true })
  const base64Data = dataUrl.split(',')[1]

  // Save to cache first
  const savedFile = await Filesystem.writeFile({
    path: 'map.png',
    data: base64Data,
    directory: Directory.Cache
  })

  // Create "GPS Tracker" album if it doesn't exist
  const { albums } = await Media.getAlbums()
  let album = albums.find(a => a.name === 'GPS Tracker')
  if (!album) {
    await Media.createAlbum({ name: 'GPS Tracker' })
    const { albums: newAlbums } = await Media.getAlbums()
    album = newAlbums.find(a => a.name === 'GPS Tracker')
  }

  // Save to gallery
  await Media.savePhoto({
    path: savedFile.uri,
    albumIdentifier: album.identifier,
    fileName: 'map'
  })

  alert('Saved to gallery!')
}

// Called when "+" button is pressed
function dropMarker(){
  markedPoints.value = [...markedPoints.value, {id: markerId, coordinates: currentPosition.value }]
  markerId++
}

// Called when "reset" button is pressed
function reset(){
  pathPoints.value = []
  markedPoints.value = []
  startMarker.value = null
  endMarker.value = null
}

</script>

<template>
  <div class="container">
    <h1> GPS Path Tracker </h1>

    <!-- Showing the map -->
    <div class="map-wrapper" id="map">
      <l-map :zoom="zoom" :center="currentPosition">
        <l-tile-layer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"/>
        <!-- Draw line for the path taken by user -->
        <l-polyline :lat-lngs="pathPoints" color="red" />

        <!-- Start marker (green) -->
        <l-circle-marker v-if="startMarker" :lat-lng="startMarker" :radius="8" color="green" />

        <!-- End marker (red) -->
        <l-circle-marker v-if="endMarker" :lat-lng="endMarker" :radius="8" color="red" />

        <!-- Drop marker(s) (blue) -->
        <l-circle-marker
          v-for="marker in markedPoints"
          :key="marker.id"
          :lat-lng="marker.coordinates"
          :radius="8"
          color="blue"
        />
      </l-map>
    </div>

    <!-- Buttons -->
    <div class="buttons">
      <button @click="startTracking" :disabled="isTracking">Start</button>
      <button @click="dropMarker">+</button>
      <button @click="stopTracking" :disabled="!isTracking">Stop</button>
      <button @click="reset" :disabled="isTracking">Reset</button>
    </div>

  </div>
  
</template>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
}

.map-wrapper {
  width: 100%;
  height: 400px;
}
</style>