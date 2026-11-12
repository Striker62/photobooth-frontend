<template>
  <q-card flat class="q-mr-md q-mb-md" style="width: min(100%, 32rem)">
    <q-card-section>
      <div class="text-h6">Belichtungsmesser</div>
      <q-btn flat :label="stream ? 'Messung stoppen' : 'Messung starten'" @click="stream ? stop() : start()" />
      <div v-if="message" role="status" class="q-mt-sm">{{ message }}</div>
      <canvas v-show="measurement" ref="preview" class="full-width q-mt-sm" aria-label="Kamera-Livevorschau" />
      <template v-if="measurement">
        <div class="row no-wrap items-end q-mt-sm" style="height: 5rem; gap: 2px" role="img" aria-label="Helligkeitshistogramm von dunkel nach hell">
          <div v-for="(height, index) in measurement.bars" :key="index" class="col bg-primary" :style="{ height: height + '%' }" />
        </div>
        <div class="row justify-between text-caption"><span>Dunkel</span><span>Hell</span></div>
        <div class="q-mt-sm">Sehr dunkel {{ measurement.dark }} % · Sehr hell {{ measurement.bright }} %</div>
        <div class="text-caption">Gemessen an der Livevorschau</div>
        <div v-for="item in metrics" :key="item.label">{{ item.label }}: {{ item.value }} {{ item.unit }}</div>
      </template>
      <AdminLightSettings
        v-if="camera?.backend_type === 'Picamera2'"
        :key="cameraIndex"
        :camera-index="cameraIndex"
        :red="measurement ? stats?.colour_gain_red : undefined"
        :blue="measurement ? stats?.colour_gain_blue : undefined"
      />
    </q-card-section>
  </q-card>
</template>

<script setup lang="ts">
import AdminLightSettings from '@/components/AdminLightSettings.vue'
import { computed, onUnmounted, ref, shallowRef, watch } from 'vue'
import { useConfigurationStore } from '@/stores/configuration-store'
import { useMainStore } from '@/stores/main-store'

const config = useConfigurationStore()
const store = useMainStore()
const cameraIndex = computed(() => config.configuration.cameras.index_backend_video)
const preview = ref<HTMLCanvasElement | null>(null)
const stream = shallowRef<WebSocket | null>(null)
const measurement = ref<{ bars: number[]; dark: string; bright: string } | null>(null)
const message = ref('')
let nextFrame: ReturnType<typeof setTimeout> | undefined

const camera = computed(() => config.configuration.cameras.group_backends.filter((entry) => entry.enabled)[cameraIndex.value]?.backend_config)
const stats = computed(() => {
  const selectedCamera = camera.value
  if (selectedCamera?.backend_type !== 'Picamera2' || !('camera_num' in selectedCamera)) return undefined
  return store.information_interval.backends[`Picamera2Backend:${selectedCamera.camera_num}`]
})
const metrics = computed(() => {
  return [
    { label: 'Belichtungszeit', value: stats.value?.exposure_time_ms, unit: 'ms' },
    { label: 'Analoge Verstärkung', value: stats.value?.again, unit: '×' },
    { label: 'Weißabgleich Rot', value: stats.value?.colour_gain_red, unit: '×' },
    { label: 'Weißabgleich Blau', value: stats.value?.colour_gain_blue, unit: '×' },
    { label: 'Farbtemperatur (geschätzt)', value: stats.value?.colour_temperature, unit: 'K' },
  ].filter((item) => typeof item.value === 'number')
})

function stop() {
  clearTimeout(nextFrame)
  stream.value?.close()
  stream.value = null
  measurement.value = null
  message.value = ''
}

function start() {
  const protocol = location.protocol === 'https:' ? 'wss' : 'ws'
  const socket = new WebSocket(`${protocol}://${location.host}/api/aquisition/stream?index_device=${cameraIndex.value}&index_subdevice=0`)
  stream.value = socket
  message.value = 'Warte auf das Kamerabild …'
  socket.onclose = () => {
    if (stream.value !== socket) return
    stop()
    message.value = 'Kein Kamerabild verfügbar. Prüfe die Livevorschau und starte die Messung erneut.'
  }
  socket.onmessage = async (event: MessageEvent<Blob>) => {
    try {
      const bitmap = await createImageBitmap(event.data)
      try {
        if (stream.value !== socket || !preview.value) return
        const canvas = preview.value
        const height = Math.max(1, Math.round((320 * bitmap.height) / bitmap.width))
        if (canvas.width !== 320 || canvas.height !== height) {
          canvas.width = 320
          canvas.height = height
        }
        const ctx = canvas.getContext('2d', { willReadFrequently: true })
        if (!ctx) throw new Error('Canvas unavailable')
        ctx.drawImage(bitmap, 0, 0, canvas.width, canvas.height)
        const pixels = ctx.getImageData(0, 0, canvas.width, canvas.height).data
        const bins = Array<number>(32).fill(0)
        for (let i = 0; i < pixels.length; i += 4) {
          const luma = 0.2126 * pixels[i]! + 0.7152 * pixels[i + 1]! + 0.0722 * pixels[i + 2]!
          const bin = Math.min(31, Math.floor(luma / 8))
          bins[bin] = bins[bin]! + 1
        }
        const max = Math.max(...bins)
        const percent = (count: number) => ((400 * count) / pixels.length).toFixed(1)
        measurement.value = { bars: bins.map((count) => (100 * count) / max), dark: percent(bins[0]!), bright: percent(bins[31]!) }
        message.value = ''
      } finally {
        bitmap.close()
      }
      nextFrame = setTimeout(() => {
        if (stream.value === socket && socket.readyState === WebSocket.OPEN) socket.send('ready')
      }, 250)
    } catch {
      if (stream.value !== socket) return
      stop()
      message.value = 'Das Kamerabild konnte nicht ausgewertet werden.'
    }
  }
}

watch(cameraIndex, stop)
onUnmounted(stop)
</script>
