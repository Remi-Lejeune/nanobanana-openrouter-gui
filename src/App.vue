<script setup>
import { ref, onMounted, onUpdated, nextTick } from 'vue'

const apiKey = ref('')
const prompt = ref('')
const inputImage = ref(null)
const inputImagePreview = ref(null)
const fileInputRef = ref(null)
const generatedImage = ref(null)
const imageHistory = ref([])
const isLoading = ref(false)
const error = ref(null)
const selectedModel = ref('nano')
const temperature = ref(1.0)
const isDarkMode = ref(true)
const inputColumnRef = ref(null)
const outputColumnRef = ref(null)

const models = {
  nano: 'google/gemini-2.5-flash-image',
  pro: 'google/gemini-3-pro-image-preview'
}


function handleImageUpload(event) {
  const file = event.target.files[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = (e) => {
    inputImagePreview.value = e.target.result
    inputImage.value = e.target.result
  }
  reader.readAsDataURL(file)
}

function clearInputImage() {
  inputImage.value = null
  inputImagePreview.value = null
  if (fileInputRef.value) {
    fileInputRef.value.value = ''
  }
}

function triggerFileInput() {
  fileInputRef.value?.click()
}

async function generateImage() {
  if (!apiKey.value) {
    error.value = 'Please enter your OpenRouter API key'
    return
  }
  if (!prompt.value) {
    error.value = 'Please enter a prompt'
    return
  }

  error.value = null
  isLoading.value = true
  generatedImage.value = null

  try {
    const messages = [
      {
        role: 'user',
        content: []
      }
    ]

    if (inputImage.value) {
      messages[0].content.push({
        type: 'image_url',
        image_url: {
          url: inputImage.value
        }
      })
    }

    messages[0].content.push({
      type: 'text',
      text: prompt.value
    })

    const response = await fetch('https://openrouter.ai/api/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiKey.value}`,
        'Content-Type': 'application/json',
        'HTTP-Referer': window.location.origin,
        'X-Title': 'nanobanana-openrouter-gui'
      },
      body: JSON.stringify({
        model: models[selectedModel.value],
        messages: messages,
        temperature: parseFloat(temperature.value)
      })
    })

    if (!response.ok) {
      const errorData = await response.json()
      throw new Error(errorData.error?.message || `HTTP error ${response.status}`)
    }

    const data = await response.json()
    const message = data.choices?.[0]?.message

    // OpenRouter returns images in message.images array
    let newImage = null
    if (message?.images && message.images.length > 0) {
      const imageData = message.images[0]
      if (imageData.image_url?.url) {
        newImage = imageData.image_url.url
      } else if (typeof imageData === 'string') {
        newImage = imageData
      }
    } else if (message?.content) {
      // Fallback: check if content contains base64 image
      const content = message.content
      if (typeof content === 'string' && content.startsWith('data:image')) {
        newImage = content
      }
    }

    if (newImage) {
      generatedImage.value = newImage
      imageHistory.value.unshift(newImage)
    }
  } catch (err) {
    error.value = err.message
  } finally {
    isLoading.value = false
  }
}

async function copyImage() {
  if (!generatedImage.value) return

  try {
    const response = await fetch(generatedImage.value)
    const blob = await response.blob()
    await navigator.clipboard.write([
      new ClipboardItem({ [blob.type]: blob })
    ])
  } catch (err) {
    error.value = 'Failed to copy image'
  }
}

function downloadImage() {
  if (!generatedImage.value) return

  const link = document.createElement('a')
  link.href = generatedImage.value
  link.download = `generated-${Date.now()}.png`
  link.click()
}

function reuseImage() {
  if (!generatedImage.value) return

  inputImage.value = generatedImage.value
  inputImagePreview.value = generatedImage.value
}

function selectFromHistory(image) {
  generatedImage.value = image
}

function toggleDarkMode() {
  isDarkMode.value = !isDarkMode.value
}

function syncColumnHeights() {
  if (inputColumnRef.value && outputColumnRef.value) {
    outputColumnRef.value.style.maxHeight = inputColumnRef.value.offsetHeight + 'px'
  }
}

onMounted(() => {
  syncColumnHeights()
  window.addEventListener('resize', syncColumnHeights)
})

onUpdated(() => {
  nextTick(syncColumnHeights)
})
</script>

<template>
  <div class="app-wrapper" :class="{ dark: isDarkMode }">
    <button type="button" class="dark-mode-btn" @click="toggleDarkMode" :title="isDarkMode ? 'Light mode' : 'Dark mode'">
      <svg v-if="isDarkMode" xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <circle cx="12" cy="12" r="5"></circle>
        <line x1="12" y1="1" x2="12" y2="3"></line>
        <line x1="12" y1="21" x2="12" y2="23"></line>
        <line x1="4.22" y1="4.22" x2="5.64" y2="5.64"></line>
        <line x1="18.36" y1="18.36" x2="19.78" y2="19.78"></line>
        <line x1="1" y1="12" x2="3" y2="12"></line>
        <line x1="21" y1="12" x2="23" y2="12"></line>
        <line x1="4.22" y1="19.78" x2="5.64" y2="18.36"></line>
        <line x1="18.36" y1="5.64" x2="19.78" y2="4.22"></line>
      </svg>
      <svg v-else xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"></path>
      </svg>
    </button>
    <div class="app">
      <h1>🍌 Nano Banana GUI (OpenRouter)</h1>

    <div class="layout">
      <div ref="inputColumnRef" class="column input-column">
        <h2>Input Image</h2>
        <div class="image-box">
          <img v-if="inputImagePreview" :src="inputImagePreview" alt="Input preview" />
          <div v-else class="placeholder">
            <span>Input</span>
          </div>
        </div>
        <div class="file-actions">
          <input
            ref="fileInputRef"
            type="file"
            accept="image/*"
            @change="handleImageUpload"
            hidden
          />
          <button type="button" class="file-btn" @click="triggerFileInput">
            Choose File
          </button>
          <button
            type="button"
            class="file-btn clear-btn"
            @click="clearInputImage"
            :disabled="!inputImagePreview"
          >
            Clear
          </button>
        </div>

        <div class="form-group">
          <label for="apiKey">OpenRouter API Key</label>
          <input
            id="apiKey"
            v-model="apiKey"
            type="text"
            placeholder="sk-or-..."
          />
        </div>

        <div class="form-group">
          <label for="prompt">Prompt</label>
          <textarea
            id="prompt"
            v-model="prompt"
            placeholder="Describe the image you want to generate..."
            rows="4"
          ></textarea>
        </div>

        <div class="form-group">
          <label>Model</label>
          <div class="model-buttons">
            <button
              type="button"
              :class="['model-btn', { active: selectedModel === 'nano' }]"
              @click="selectedModel = 'nano'"
            >
              Nano Banana
            </button>
            <button
              type="button"
              :class="['model-btn', { active: selectedModel === 'pro' }]"
              @click="selectedModel = 'pro'"
            >
              Nano Banana Pro
            </button>
          </div>
        </div>

        <div class="form-group">
          <label for="temperature">Temperature: {{ temperature }}</label>
          <input
            id="temperature"
            v-model="temperature"
            type="range"
            min="0"
            max="2"
            step="0.1"
            class="temperature-slider"
          />
          <div class="temperature-labels">
            <span>Predictable</span>
            <span>Creative</span>
          </div>
        </div>

        <button class="generate-btn" @click="generateImage" :disabled="isLoading">
          {{ isLoading ? 'Generating...' : 'Generate Image' }}
        </button>

        <div v-if="error" class="error">
          {{ error }}
        </div>
      </div>

      <div ref="outputColumnRef" class="column output-column">
        <h2>Generated Image</h2>
        <div class="image-box">
          <img v-if="generatedImage" :src="generatedImage" alt="Generated image" />
          <div v-else class="placeholder">
            <span>Output</span>
          </div>
        </div>
        <div v-if="generatedImage" class="image-actions">
          <button type="button" class="action-btn" @click="copyImage">Copy</button>
          <button type="button" class="action-btn" @click="downloadImage">Download</button>
          <button type="button" class="action-btn" @click="reuseImage">Reuse</button>
        </div>

        <div v-if="imageHistory.length > 0" class="history">
          <h3>History</h3>
          <div class="history-grid">
            <div
              v-for="(image, index) in imageHistory"
              :key="index"
              class="history-item"
              :class="{ active: image === generatedImage }"
              @click="selectFromHistory(image)"
            >
              <img :src="image" :alt="'History ' + (index + 1)" />
            </div>
          </div>
        </div>
      </div>
    </div>
    </div>
  </div>
</template>

<style scoped>
.app-wrapper {
  min-height: 100vh;
  background: #ffffff;
  color: #111827;
  transition: background 0.2s, color 0.2s;
  font-family: system-ui, -apple-system, sans-serif;
  position: relative;
}

.app-wrapper.dark {
  background: #111827;
  color: #f9fafb;
}

.app {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
  padding-top: 4rem;
}

h1 {
  text-align: center;
  margin-bottom: 2rem;
}

.dark-mode-btn {
  position: fixed;
  top: 1rem;
  right: 1rem;
  padding: 0.5rem;
  background: #374151;
  color: white;
  border: none;
  border-radius: 50%;
  cursor: pointer;
  transition: background 0.15s;
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
}

.dark-mode-btn:hover {
  background: #1f2937;
}

.app-wrapper.dark .dark-mode-btn {
  background: #e5e7eb;
  color: #111827;
}

.app-wrapper.dark .dark-mode-btn:hover {
  background: #f9fafb;
}

h2 {
  margin-bottom: 1rem;
  font-size: 1.25rem;
}

.layout {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 2rem;
  align-items: start;
}

.column {
  min-width: 0;
}

.output-column {
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.image-box {
  aspect-ratio: 1;
  border: 2px dashed #d1d5db;
  border-radius: 8px;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f9fafb;
}

.image-box img {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.placeholder {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 100%;
  color: #9ca3af;
  font-size: 1.5rem;
  font-weight: 500;
}

.file-actions {
  display: flex;
  gap: 0.5rem;
  margin-top: 1rem;
  margin-bottom: 1.5rem;
}

.file-btn {
  flex: 1;
  padding: 0.6rem 1rem;
  background: #4f46e5;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 0.875rem;
  cursor: pointer;
  transition: background 0.15s;
}

.file-btn:hover:not(:disabled) {
  background: #4338ca;
}

.file-btn.clear-btn {
  background: #6b7280;
}

.file-btn.clear-btn:hover:not(:disabled) {
  background: #4b5563;
}

.file-btn:disabled {
  background: #d1d5db;
  cursor: not-allowed;
}

.form-group {
  margin-bottom: 1.5rem;
}

label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 600;
}

input[type="text"],
input[type="password"],
textarea {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 1rem;
  box-sizing: border-box;
}

textarea {
  resize: vertical;
}

.model-buttons {
  display: flex;
  gap: 0.5rem;
}

.model-btn {
  flex: 1;
  padding: 0.75rem;
  background: #e5e7eb;
  color: #374151;
  border: 2px solid transparent;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
}

.model-btn:hover {
  background: #d1d5db;
}

.model-btn.active {
  background: #4f46e5;
  color: white;
  border-color: #4f46e5;
}

.temperature-slider {
  width: 100%;
  height: 6px;
  border-radius: 3px;
  background: #e5e7eb;
  outline: none;
  -webkit-appearance: none;
  appearance: none;
  cursor: pointer;
}

.temperature-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #4f46e5;
  cursor: pointer;
}

.temperature-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #4f46e5;
  cursor: pointer;
  border: none;
}

.temperature-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #6b7280;
  margin-top: 0.25rem;
}

.generate-btn {
  width: 100%;
  padding: 1rem;
  background: #4f46e5;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
}

.generate-btn:hover:not(:disabled) {
  background: #4338ca;
}

.generate-btn:disabled {
  background: #9ca3af;
  cursor: not-allowed;
}

.error {
  margin-top: 1rem;
  padding: 1rem;
  background: #fee2e2;
  color: #dc2626;
  border-radius: 4px;
}

.image-actions {
  display: flex;
  gap: 0.5rem;
  margin-top: 1rem;
}

.action-btn {
  flex: 1;
  padding: 0.5rem 1rem;
  background: #374151;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 0.875rem;
  cursor: pointer;
}

.action-btn:hover {
  background: #1f2937;
}

.history {
  margin-top: 1.5rem;
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.history h3 {
  font-size: 1rem;
  margin-bottom: 0.75rem;
  color: #374151;
  flex-shrink: 0;
}

.history-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  grid-auto-rows: min-content;
  gap: 0.5rem;
  overflow-y: auto;
  flex: 1;
  padding-right: 0.25rem;
}

.history-grid::-webkit-scrollbar {
  width: 6px;
}

.history-grid::-webkit-scrollbar-track {
  background: transparent;
}

.history-grid::-webkit-scrollbar-thumb {
  background: #d1d5db;
  border-radius: 3px;
}

.history-grid::-webkit-scrollbar-thumb:hover {
  background: #9ca3af;
}

.history-item {
  aspect-ratio: 1;
  border: 2px solid #e5e7eb;
  border-radius: 4px;
  overflow: hidden;
  cursor: pointer;
  transition: border-color 0.15s;
}

.history-item:hover {
  border-color: #9ca3af;
}

.history-item.active {
  border-color: #4f46e5;
}

.history-item img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

@media (max-width: 768px) {
  .layout {
    grid-template-columns: 1fr;
  }
}

/* Dark mode styles */
.app-wrapper.dark .image-box {
  border-color: #374151;
  background: #1f2937;
}

.app-wrapper.dark .placeholder {
  color: #6b7280;
}

.app-wrapper.dark input[type="text"],
.app-wrapper.dark input[type="password"],
.app-wrapper.dark textarea {
  background: #1f2937;
  border-color: #374151;
  color: #f9fafb;
}

.app-wrapper.dark input[type="text"]::placeholder,
.app-wrapper.dark textarea::placeholder {
  color: #6b7280;
}

.app-wrapper.dark .model-btn {
  background: #374151;
  color: #e5e7eb;
}

.app-wrapper.dark .model-btn:hover {
  background: #4b5563;
}

.app-wrapper.dark .model-btn.active {
  background: #4f46e5;
  color: white;
}

.app-wrapper.dark .temperature-slider {
  background: #374151;
}

.app-wrapper.dark .temperature-labels {
  color: #9ca3af;
}

.app-wrapper.dark .error {
  background: #7f1d1d;
  color: #fecaca;
}

.app-wrapper.dark .history h3 {
  color: #d1d5db;
}

.app-wrapper.dark .history-item {
  border-color: #374151;
}

.app-wrapper.dark .history-item:hover {
  border-color: #6b7280;
}

.app-wrapper.dark .history-grid::-webkit-scrollbar-thumb {
  background: #4b5563;
}

.app-wrapper.dark .history-grid::-webkit-scrollbar-thumb:hover {
  background: #6b7280;
}

.app-wrapper.dark .action-btn {
  background: #4b5563;
}

.app-wrapper.dark .action-btn:hover {
  background: #6b7280;
}

.app-wrapper.dark .file-btn.clear-btn {
  background: #4b5563;
}

.app-wrapper.dark .file-btn.clear-btn:hover:not(:disabled) {
  background: #6b7280;
}

.app-wrapper.dark .file-btn:disabled {
  background: #374151;
  color: #6b7280;
}
</style>
