<template>
  <q-card flat class="q-mr-md q-mb-md" style="width: min(100%, 32rem)">
    <q-card-section>
      <div class="text-h6">Druckwarteschlange</div>
      <q-select v-model="queue" :options="queues" label="Drucker" :disable="busy" @update:model-value="refresh" />
      <q-banner v-if="error" class="text-negative">{{ error }}</q-banner>
      <div v-if="data" class="q-mt-md">
        <div>{{ labels[data.state] ?? 'Status unbekannt' }} · {{ data.jobs.length }} offene Aufträge</div>
        <div class="text-caption q-my-sm" style="white-space: pre-wrap; overflow-wrap: anywhere">{{ data.detail }}</div>
        <q-list separator>
          <q-item v-for="job in data.jobs" :key="job.id">
            <q-item-section>
              <q-item-label>{{ job.id }}</q-item-label>
              <q-item-label caption>{{ job.submitted }} · {{ job.owner }}</q-item-label>
            </q-item-section>
            <q-item-section side>
              <q-btn flat color="negative" label="Abbrechen" :disable="busy" @click="confirm = { job: job.id }" />
            </q-item-section>
          </q-item>
        </q-list>
      </div>
    </q-card-section>
    <q-card-actions>
      <q-btn flat label="Aktualisieren" :loading="busy" @click="refresh" />
      <q-btn
        v-if="data?.state === 'ok' || data?.state === 'disabled'"
        flat
        label="Warteschlange neu starten"
        :disable="busy"
        @click="confirm = { restart: true }"
      />
      <q-btn v-if="data?.state === 'disabled'" flat label="Fortsetzen" :disable="busy" @click="confirm = {}" />
    </q-card-actions>
    <q-dialog :model-value="confirm !== null" @hide="confirm = null">
      <q-card>
        <q-card-section>
          {{ confirm?.job ? `Druckauftrag ${confirm.job} abbrechen?` : confirm?.restart
          ? 'Warteschlange neu starten? Offene Aufträge können danach sofort gedruckt werden.'
          : 'Warteschlange fortsetzen? Offene Aufträge können sofort gedruckt werden.' }}
        </q-card-section>
        <q-card-actions align="right">
          <q-btn flat label="Zurück" v-close-popup />
          <q-btn flat color="primary" label="Bestätigen" @click="apply" />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </q-card>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue'
import { _fetch } from '@/util/fetch_api.js'

interface QueueStatus {
  state: string
  detail: string
  jobs: { id: string; owner: string; submitted: string }[]
}

const labels: Record<string, string> = { ok: 'Bereit', busy: 'Druckt', disabled: 'Angehalten' }
const queues = ref<string[]>([])
const queue = ref<string | null>(null)
const data = ref<QueueStatus | null>(null)
const busy = ref(false)
const error = ref('')
const confirm = ref<{ job?: string; restart?: boolean } | null>(null)
const endpoint = '/api/admin/printers'

async function request(path: string, method = 'GET') {
  const response = await _fetch(endpoint + path, { method })
  if (!response.ok) {
    const payload = await response.json().catch(() => null)
    throw new Error(payload?.detail || `CUPS-Abfrage fehlgeschlagen (${response.status})`)
  }
  return response.json()
}

async function refresh() {
  busy.value = true
  error.value = ''
  data.value = null
  try {
    queues.value = await request('')
    if (!queue.value || !queues.value.includes(queue.value)) {
      queue.value = queues.value.find((name) => name === 'Selphy_CP1500_USB') ?? queues.value[0] ?? null
    }
    if (queue.value) data.value = await request('/' + encodeURIComponent(queue.value))
    else error.value = 'Keine lokale Druckwarteschlange gefunden.'
  } catch (cause) {
    error.value = String(cause)
  } finally {
    busy.value = false
  }
}

async function apply() {
  if (!queue.value || !confirm.value) return
  const path = '/' + encodeURIComponent(queue.value)
  const action = confirm.value.job ? '/jobs/' + encodeURIComponent(confirm.value.job) + '/cancel' : confirm.value.restart ? '/restart' : '/resume'
  confirm.value = null
  busy.value = true
  error.value = ''
  try {
    data.value = await request(path + action, 'POST')
  } catch (cause) {
    data.value = await request(path).catch(() => null)
    error.value = String(cause)
  } finally {
    busy.value = false
  }
}

onMounted(refresh)
</script>
