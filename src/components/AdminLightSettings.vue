<template>
  <div class="q-mt-md">
    <q-btn v-if="!draft" flat label="Licht einstellen" :loading="busy" @click="load" />
    <q-banner v-if="error" class="text-negative" role="alert">{{ error }}</q-banner>
    <div v-if="notice" role="status" class="q-mt-sm">{{ notice }}</div>
    <template v-if="draft">
      <q-select v-if="profiles.length" v-model="selected" :options="profiles.map((profile) => profile.name)" label="Lichtprofil laden" :disable="busy" @update:model-value="loadProfile" />
      <q-input v-model.number="draft.exposure_compensation" type="number" min="-3" max="3" step="0.1" label="Belichtungskorrektur (EV)" :disable="busy" />
      <q-select v-model="draft.white_balance_mode" :options="modes" label="Weißabgleich" :disable="busy" />
      <template v-if="draft.white_balance_mode === 'Auto'">
        <div class="q-mt-sm">Blaukorrektur: {{ draft.auto_blue_correction }} %</div>
        <q-slider v-model="draft.auto_blue_correction" :min="-15" :max="15" :step="1" :disable="busy" aria-label="Blaukorrektur" />
        <div class="row justify-between text-caption"><span>Wärmer</span><span>Kühler</span></div>
      </template>
      <template v-if="draft.white_balance_mode === 'Manual'">
        <q-input v-model.number="draft.white_balance_red_gain" type="number" min="0.001" max="32" step="0.001" label="Rot-Verstärkung" :disable="busy" />
        <q-input v-model.number="draft.white_balance_blue_gain" type="number" min="0.001" max="32" step="0.001" label="Blau-Verstärkung" :disable="busy" />
      </template>
      <q-btn flat label="Weißabgleich aus Vorschau übernehmen" :disable="busy || !validGains" @click="copyGains" />
      <div v-if="testing" class="q-mt-sm text-caption">Testwerte aktiv. Du kannst zur Fotobox wechseln und ein Foto machen.</div>
      <div class="row q-mt-sm">
        <q-btn flat label="Vorübergehend testen" :disable="busy || !validDraft" @click="trySettings" />
        <q-btn v-if="testing" flat label="Test beenden" :disable="busy" @click="endTrial" />
        <q-btn color="primary" flat label="Kameraeinstellungen speichern" :disable="busy || !validDraft" @click="save" />
      </div>
      <q-input v-model="profileName" maxlength="60" label="Name für neues Lichtprofil" :disable="busy" />
      <q-btn flat label="Profil speichern" :disable="busy || !validDraft || !profileName.trim()" @click="saveProfile" />
    </template>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { _fetch } from '@/util/fetch_api.js'

const props = defineProps<{ cameraIndex: number; red: unknown; blue: unknown }>()
const modes = ['Daylight', 'Auto', 'Cloudy', 'Indoor', 'Tungsten', 'Fluorescent', 'Manual'] as const
interface LightSettings {
  exposure_compensation: number
  auto_blue_correction: number
  white_balance_mode: (typeof modes)[number]
  white_balance_red_gain: number
  white_balance_blue_gain: number
}
type Profile = LightSettings & { name: string }
interface LightResponse { settings: LightSettings; profiles: Profile[]; trial: LightSettings | null }

const draft = ref<LightSettings | null>(null)
const profiles = ref<Profile[]>([])
const testing = ref(false)
const selected = ref<string | null>(null)
const profileName = ref('')
const busy = ref(false)
const error = ref('')
const notice = ref('')
const endpoint = computed(() => '/api/admin/config/light/' + props.cameraIndex)

function isGain(value: unknown): value is number {
  return typeof value === 'number' && value > 0 && value <= 32
}

const validGains = computed(() => isGain(props.red) && isGain(props.blue))
const validDraft = computed(() => {
  const value = draft.value
  return value && Number.isFinite(value.exposure_compensation) &&
    Math.abs(value.exposure_compensation) <= 3 &&
    isGain(value.white_balance_red_gain) && isGain(value.white_balance_blue_gain)
})

async function request(path = '', method = 'GET', body?: unknown) {
  const response = await _fetch(endpoint.value + path, {
    method, headers: { 'Content-Type': 'application/json' },
    ...(body !== undefined && { body: JSON.stringify(body) }),
  })
  if (!response.ok) {
    const payload = await response.json().catch(() => null)
    throw new Error(typeof payload?.detail === 'string' ? payload.detail : 'Kameraeinstellungen konnten nicht übernommen werden.')
  }
  return response.json()
}

async function perform(action: () => Promise<void>) {
  busy.value = true
  error.value = ''
  notice.value = ''
  try {
    await action()
  } catch (cause) {
    error.value = cause instanceof Error ? cause.message : String(cause)
  } finally {
    busy.value = false
  }
}

function show(data: LightResponse) {
  draft.value = { ...(data.trial ?? data.settings), auto_blue_correction: (data.trial ?? data.settings).auto_blue_correction ?? 0 }
  profiles.value = data.profiles
  testing.value = data.trial !== null
}

function load() { return perform(async () => show(await request())) }

function copyGains() {
  if (!draft.value || !isGain(props.red) || !isGain(props.blue)) return
  draft.value.white_balance_mode = 'Manual'
  draft.value.white_balance_red_gain = props.red
  draft.value.white_balance_blue_gain = props.blue
  notice.value = 'Rot- und Blauwerte in den Entwurf übernommen.'
}

function loadProfile(name: string) {
  const profile = profiles.value.find((item) => item.name === name)
  if (!profile) return
  const { name: profileLabel, ...settings } = profile
  draft.value = { ...settings, auto_blue_correction: settings.auto_blue_correction ?? 0 }
  profileName.value = profileLabel
}

function trySettings() {
  return perform(async () => {
    await request('/trial', 'PUT', draft.value)
    testing.value = true
  })
}

function endTrial() {
  return perform(async () => {
    await request('/trial', 'PUT', null)
    show(await request())
  })
}

function save() {
  return perform(async () => {
    show(await request('', 'PUT', draft.value))
    notice.value = 'Kameraeinstellungen gespeichert.'
  })
}

function saveProfile() {
  const name = profileName.value.trim()
  if (profiles.value.some((profile) => profile.name === name) && !window.confirm('Dieses Lichtprofil überschreiben?')) return
  return perform(async () => {
    const data: LightResponse = await request('/profiles', 'POST', { ...draft.value, name })
    profiles.value = data.profiles
    selected.value = name
    notice.value = 'Lichtprofil gespeichert.'
  })
}

onMounted(load)
</script>
