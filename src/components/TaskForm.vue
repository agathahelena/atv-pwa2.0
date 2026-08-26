```vue
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
        </form>
      <form class="task-form" @submit.prevent="handleGetLocation">
     <div class="location-section">
  <button
    type="button"
    class="location-button"
    :disabled="loadingLocation"
    @click="handleGetLocation"
  >
    📍
    {{ loadingLocation ? 'Obtendo localização...' : 'Usar localização atual' }}
  </button>

  <button
    v-if="location"
    type="button"
    class="location-clear"
    @click="clearLocation"
  >
    Remover localização
  </button>

  <p v-if="locationError" class="location-error">
    {{ locationError }}
  </p>

  <div v-if="location" class="location-info">
    <p>
      📍 Localização capturada
    </p>

    <p v-if="location.accuracy != null">
      Precisão: {{ accuracyLevel }}
      ({{ Math.round(location.accuracy) }} m)
    </p>

    <p v-if="permissionState === 'granted'">
      Permissão concedida
    </p>
  </div>
</div>
      </form>
</template>

<script setup>
import { ref, watch, computed } from 'vue'

import tasksApi from '../api/tasksApi.js'
import geocodingApi from '../api/geocodingApi.js'

import { useGeolocation } from '../composables/useGeolocation.js'

import {
  classifyAccuracy,
  buildLocationPayload,
} from '../utils/location.js'

const {
  location,
  loadingLocation,
  locationError,
  permissionState,
  requestCurrentLocation,
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
  const currentLocation = await requestCurrentLocation()

  if (!currentLocation) return

  try {
    const address = await geocodingApi.reverse(
      currentLocation.latitude,
      currentLocation.longitude,
    )

    if (address?.label) {
      setLocationLabel(address.label)
    }
  } catch (err) {
    console.error('Erro ao obter endereço:', err)
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
  transition: border-color 0.2s;
}

.task-button-cancel:hover {
  border-color: #aaa;
}

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
  transition: background-color 0.2s;
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
  margin: 0;
  flex-basis: 100%;
}
.accuracy-badge {
  font-size: 0.75rem;
  padding: 2px 8px;
  border-radius: 12px;
}
.accuracy-badge--boa { background: #d4edda; color: #155724; }
.accuracy-badge--moderada { background: #fff3cd; color: #856404; }
.accuracy-badge--baixa { background: #f8d7da; color: #721c24; }
</style>

