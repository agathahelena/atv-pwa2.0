<template>
  <form class="task-form" @submit.prevent="handleSubmit">
    <div class="task-row">
      <input
        v-model="newTask"
        type="text"
        placeholder="Nova tarefa..."
        class="task-input"
      />

      <button
        type="submit"
        class="task-button"
        :disabled="uploading"
      >
        {{ editingTask ? 'Alterar' : 'Adicionar' }}
      </button>

      <button
        v-if="editingTask"
        type="button"
        class="task-button-cancel"
        @click="handleCancel"
      >
        Cancelar
      </button>
    </div>

    <!-- IMAGEM -->
    <div class="image-section">
      <img
        v-if="previewUrl || editingTask?.img_url"
        :src="previewUrl || editingTask?.img_url"
        class="image-preview"
        alt="Imagem da tarefa"
      />

      <label
        class="image-label"
        :class="{ disabled: uploading }"
      >
        <span v-if="uploading" class="upload-status">
          Enviando...
        </span>

        <span v-else>
          {{
            previewUrl || editingTask?.img_url
              ? 'Trocar imagem'
              : isMobileDevice
                ? 'Fotografar'
                : 'Adicionar imagem'
          }}
        </span>

        <input
          type="file"
          accept="image/jpeg,image/png"
          capture="environment"
          class="image-input"
          :disabled="uploading"
          @change="handleImageChange"
        />
      </label>
    </div>

    <p class="image-help">
      Em celular, o botão pode abrir a câmera.
      Em notebook, abre o seletor de arquivos.
    </p>

    <!-- LOCALIZAÇÃO -->
    <div class="location-section">
      <div class="location-actions">
        <button
          type="button"
          class="location-button"
          :disabled="loadingLocation"
          @click="handleGetLocation"
        >
          📍
          {{
            loadingLocation
              ? 'Obtendo localização...'
              : 'Usar localização atual'
          }}
        </button>

        <button
          v-if="location"
          type="button"
          class="location-clear"
          @click="clearLocation"
        >
          Remover localização
        </button>
      </div>

      <!-- MAPA -->
      <div class="map-section">
        <p class="map-help">
          Clique no mapa para escolher uma localização.
        </p>

        <TaskLocationMap
          :location="location"
          @select="handleMapLocationSelect"
        />
      </div>

      <!-- INFORMAÇÕES DA LOCALIZAÇÃO -->
      <div v-if="location" class="location-info">
        <p>
          📍
          {{ location.label || 'Localização selecionada' }}
        </p>

        <p v-if="location.accuracy != null">
          Precisão:
          <strong>{{ accuracyLevel }}</strong>
          ({{ Math.round(location.accuracy) }} m)
        </p>

        <p class="coordinates">
          Latitude: {{ location.latitude }}<br />
          Longitude: {{ location.longitude }}
        </p>
      </div>

      <p v-if="locationError" class="location-error">
        {{ locationError }}
      </p>
    </div>
  </form>
</template>

<script setup>
import { ref, watch, computed } from 'vue'

import tasksApi from '../api/tasksApi.js'
import geocodingApi from '../api/geocodingApi.js'
import TaskLocationMap from './TaskLocationMap.vue'

import { useGeolocation } from '../composables/useGeolocation.js'

import {
  classifyAccuracy,
  buildLocationPayload,
} from '../utils/location.js'

const {
  location,
  loadingLocation,
  locationError,
  requestCurrentLocation,
  setLocationFromTask,
  clearLocation,
  setLocationLabel,
} = useGeolocation()

const accuracyLevel = computed(() =>
  classifyAccuracy(location.value?.accuracy)
)

const isMobileDevice = ref(
  !window.matchMedia('(pointer: fine)').matches
)

const props = defineProps({
  editingTask: {
    type: Object,
    default: null,
  },
})

const emit = defineEmits(['add', 'update', 'cancel'])

const newTask = ref('')
const previewUrl = ref(null)
const imgAttachmentKey = ref(null)
const uploading = ref(false)

watch(
  () => props.editingTask,
  (task) => {
    newTask.value = task ? task.title : ''

    if (previewUrl.value) {
      URL.revokeObjectURL(previewUrl.value)
    }

    previewUrl.value = null
    imgAttachmentKey.value = null

    if (task) {
      setLocationFromTask(task)
    } else {
      clearLocation()
    }
  },
  { immediate: true }
)

async function handleImageChange(event) {
  const file = event.target.files[0]

  if (!file) return

  if (previewUrl.value) {
    URL.revokeObjectURL(previewUrl.value)
  }

  previewUrl.value = URL.createObjectURL(file)
  uploading.value = true

  try {
    const response = await tasksApi.uploadImage(file)

    imgAttachmentKey.value = response.data.attachment_key
  } catch (err) {
    console.error('Erro ao fazer upload da imagem', err)

    previewUrl.value = null
    imgAttachmentKey.value = null
  } finally {
    uploading.value = false
  }
}

async function handleGetLocation() {
  const captured = await requestCurrentLocation()

  if (!captured) return

  try {
    const address = await geocodingApi.reverse(
      captured.latitude,
      captured.longitude,
    )

    setLocationLabel(address?.label)
  } catch {
    locationError.value =
      'Localização obtida, mas não foi possível identificar a rua.'
  }
}

async function handleMapLocationSelect(selectedLocation) {
  location.value = selectedLocation

  try {
    const address = await geocodingApi.reverse(
      selectedLocation.latitude,
      selectedLocation.longitude,
    )

    setLocationLabel(address?.label)
  } catch {
    locationError.value =
      'Localização selecionada, mas não foi possível identificar a rua.'
  }
}

function handleSubmit() {
  if (!newTask.value.trim()) return

  const payload = {
    title: newTask.value.trim(),
    img_attachment_key: imgAttachmentKey.value,
    ...buildLocationPayload(location.value),
  }

  if (props.editingTask) {
    emit('update', props.editingTask.id, payload)
  } else {
    emit('add', payload)
  }

  newTask.value = ''

  if (previewUrl.value) {
    URL.revokeObjectURL(previewUrl.value)
  }

  previewUrl.value = null
  imgAttachmentKey.value = null

  clearLocation()
}

function handleCancel() {
  newTask.value = ''

  if (previewUrl.value) {
    URL.revokeObjectURL(previewUrl.value)
  }

  previewUrl.value = null
  imgAttachmentKey.value = null

  clearLocation()

  emit('cancel')
}
</script>

<style scoped>
.task-form {
  margin-bottom: 24px;
}

.task-row {
  display: flex;
  gap: 8px;
  margin-bottom: 12px;
}

.task-input {
  flex: 1;
  padding: 12px;
  border: 2px solid #ddd;
  border-radius: 8px;
  font-size: 1rem;
  outline: none;
  transition: border-color 0.2s;
}

.task-input:focus {
  border-color: #4a90d9;
}

.task-button {
  padding: 12px 20px;
  background-color: #4a90d9;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  cursor: pointer;
  transition: background-color 0.2s;
}

.task-button:hover:not(:disabled) {
  background-color: #357abd;
}

.task-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.task-button-cancel {
  padding: 12px 16px;
  background-color: transparent;
  color: #666;
  border: 2px solid #ddd;
  border-radius: 8px;
  font-size: 1rem;
  cursor: pointer;
}

.task-button-cancel:hover {
  border-color: #aaa;
}

/* IMAGEM */

.image-section {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 12px;
  background: #f8f9fa;
  border-radius: 8px;
  border: 1px dashed #ccc;
  flex-wrap: wrap;
}

.image-preview {
  width: 56px;
  height: 56px;
  object-fit: cover;
  border-radius: 6px;
  border: 1px solid #ddd;
  flex-shrink: 0;
}

.image-label {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 14px;
  background: white;
  border: 1.5px solid #4a90d9;
  color: #4a90d9;
  border-radius: 6px;
  font-size: 0.875rem;
  cursor: pointer;
}

.image-label:hover:not(.disabled) {
  background: #eaf2fb;
}

.image-label.disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.image-input {
  display: none;
}

.upload-status {
  color: #888;
}

.image-help {
  font-size: 0.75rem;
  color: #999;
  margin: 0 0 16px;
}

/* LOCALIZAÇÃO */

.location-section {
  margin-top: 16px;
}

.location-actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.location-button {
  padding: 10px 14px;
  border: none;
  border-radius: 8px;
  background-color: #4a90d9;
  color: white;
  cursor: pointer;
  font-size: 0.9rem;
}

.location-button:hover:not(:disabled) {
  background-color: #357abd;
}

.location-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.location-clear {
  padding: 10px 14px;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: white;
  color: #666;
  cursor: pointer;
}

.location-clear:hover {
  border-color: #aaa;
}

/* MAPA */

.map-section {
  margin-top: 12px;
}

.map-help {
  margin: 0 0 8px;
  font-size: 0.85rem;
  color: #777;
}

/* INFORMAÇÕES */

.location-info {
  margin-top: 10px;
  padding: 10px 12px;
  background: #f8f9fa;
  border-radius: 8px;
  font-size: 0.9rem;
}

.location-info p {
  margin: 4px 0;
}

.coordinates {
  font-size: 0.75rem;
  color: #888;
}

.location-error {
  margin-top: 8px;
  color: #e74c3c;
  font-size: 0.85rem;
}

.accuracy-badge {
  font-size: 0.75rem;
  padding: 2px 8px;
  border-radius: 12px;
}

.accuracy-badge--boa {
  background: #d4edda;
  color: #155724;
}

.accuracy-badge--moderada {
  background: #fff3cd;
  color: #856404;
}

.accuracy-badge--baixa {
  background: #f8d7da;
  color: #721c24;
}
</style>