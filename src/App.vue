<script setup lang="ts">
import { ref, onMounted, watch } from 'vue'
import video1 from './assets/subway.mp4'
import video2 from './assets/subway.mp4'
import video3 from './assets/subway.mp4'

const textContent = ref('')
const words = ref<string[]>([])
const currentWordIndex = ref(0)
const canvas = ref<HTMLCanvasElement | null>(null)
const videoElement = ref<HTMLVideoElement | null>(null)
const mediaRecorder = ref<MediaRecorder | null>(null)
const isRecording = ref(false)
const selectedVideo = ref(1)
const recordedChunks = ref<Blob[]>([])

const changeVideo = (videoNumber: number) => {
  selectedVideo.value = videoNumber
}

const handleFileUpload = async (event: Event) => {
  const file = (event.target as HTMLInputElement).files?.[0]
  if (!file) return
  
  if (file.type === 'application/pdf') {
    textContent.value = 'PDF content will be processed here'
  } else if (file.type === 'application/vnd.openxmlformats-officedocument.wordprocessingml.document') {
    textContent.value = 'DOCX content will be processed here'
  } else if (file.type === 'text/plain') {
    textContent.value = await file.text()
  }
  
  processText()
}

const processText = () => {
  words.value = textContent.value
    .replace(/[.,\/#!$%\^&\*;:{}=\-_`~()]/g, '')
    .split(' ')
    .filter(word => word.trim() !== '')
}

watch(textContent, () => {
  processText()
})

const getVideoSource = (number: number) => {
  switch(number) {
    case 1: return video1
    case 2: return video2
    case 3: return video3
    default: return video1
  }
}

const startRecording = async () => {
  if (!canvas.value || !videoElement.value) return
  
  isRecording.value = true
  recordedChunks.value = []
  currentWordIndex.value = 0

  // Set up canvas context
  const ctx = canvas.value.getContext('2d')
  if (!ctx) return

  // Set canvas size to match video
  canvas.value.width = videoElement.value.videoWidth
  canvas.value.height = videoElement.value.videoHeight

  // Set up media recorder with more widely supported codec
  const stream = canvas.value.captureStream(30) // 30 FPS
  try {
    mediaRecorder.value = new MediaRecorder(stream, {
      mimeType: 'video/webm'  // Removed specific codec
    })
  } catch (e) {
    // Fallback if webm is not supported
    mediaRecorder.value = new MediaRecorder(stream)
  }

  mediaRecorder.value.ondataavailable = (event) => {
    if (event.data.size > 0) {
      recordedChunks.value.push(event.data)
    }
  }

  mediaRecorder.value.onstop = () => {
    const blob = new Blob(recordedChunks.value, { type: 'video/webm' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = 'word-video.webm'
    a.click()
    URL.revokeObjectURL(url)
  }

  // Start recording
  mediaRecorder.value.start(100) // Collect 100ms chunks

  // Animation loop
  const drawFrame = () => {
    if (!isRecording.value || !ctx || !videoElement.value || !canvas.value) return

    // Draw video frame
    ctx.drawImage(videoElement.value, 0, 0, canvas.value.width, canvas.value.height)

    // Draw current word
    if (currentWordIndex.value < words.value.length) {
      ctx.font = '48px Arial'
      ctx.fillStyle = 'white'
      ctx.textAlign = 'center'
      ctx.textBaseline = 'middle'
      ctx.fillText(
        words.value[currentWordIndex.value],
        canvas.value.width / 2,
        canvas.value.height / 2
      )

      if (videoElement.value.currentTime >= (currentWordIndex.value + 1) * 1) { // 1 second per word
        currentWordIndex.value++
      }
    } else {
      stopRecording()
      return
    }

    requestAnimationFrame(drawFrame)
  }

  // Start animation loop
  videoElement.value.currentTime = 0
  videoElement.value.play()
  drawFrame()
}

const stopRecording = () => {
  if (!mediaRecorder.value || !videoElement.value) return
  
  isRecording.value = false
  mediaRecorder.value.stop()
  videoElement.value.pause()
}

onMounted(() => {
  canvas.value = document.createElement('canvas')
})
</script>

<template>
  <div class="relative min-h-screen">
    <!-- Video selection buttons -->
    <div class="absolute top-6 right-6 z-20 flex gap-3">
      <button 
        v-for="n in 3" 
        :key="n"
        @click="changeVideo(n)"
        class="px-6 py-3 rounded-lg transition-all duration-300 shadow-lg hover:scale-105"
        :class="selectedVideo === n ? 'bg-indigo-600 text-white' : 'bg-white/90 text-gray-800 hover:bg-white'"
      >
        Video {{ n }}
      </button>
    </div>

    <!-- Hidden video element for processing -->
    <video 
      ref="videoElement"
      :src="getVideoSource(selectedVideo)"
      class="hidden"
      crossorigin="anonymous"
    ></video>

    <!-- Content overlay -->
    <div class="min-h-screen bg-gradient-to-b from-gray-900 to-gray-800 py-8 flex flex-col items-center">
      <div class="w-full max-w-3xl px-6 md:px-8">
        <!-- File input -->
        <div class="mb-8">
          <label class="block mb-4 text-white text-lg font-medium">Upload File</label>
          <label class="flex items-center justify-center w-full h-32 px-4 transition bg-white/90 border-2 border-gray-300 border-dashed rounded-lg cursor-pointer hover:bg-white">
            <div class="flex flex-col items-center">
              <svg class="w-8 h-8 text-gray-600 mb-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12"/>
              </svg>
              <span class="text-gray-600">Drop files or click to upload</span>
            </div>
            <input 
              type="file" 
              accept=".pdf,.docx,.txt"
              @change="handleFileUpload"
              class="hidden"
            >
          </label>
        </div>

        <!-- Text input -->
        <div class="mb-8">
          <label class="block mb-4 text-white text-lg font-medium">Or Type Text</label>
          <textarea
            v-model="textContent"
            placeholder="Type your text here..."
            class="w-full h-40 p-4 rounded-lg bg-white/90 hover:bg-white transition-all duration-300 focus:ring-2 focus:ring-indigo-500 focus:outline-none"
          ></textarea>
        </div>

        <!-- Preview of current word (optional) -->
        <div v-if="words.length > 0" class="mb-8 text-center">
          <p class="text-white mb-2">Words to be processed: {{ words.length }}</p>
          <p class="text-white text-sm">Each word will appear for 1 second in the final video</p>
        </div>

        <!-- Controls -->
        <div class="flex justify-center">
          <button 
            @click="startRecording"
            :disabled="isRecording || words.length === 0"
            class="group relative inline-flex items-center px-8 py-4 text-lg font-medium rounded-full transition-all duration-300"
            :class="isRecording || words.length === 0 ? 'bg-gray-500 cursor-not-allowed' : 'bg-indigo-600 hover:bg-indigo-700'"
          >
            <span class="relative text-white flex items-center gap-3">
              <svg class="w-6 h-6" fill="currentColor" viewBox="0 0 20 20">
                <path d="M6.3 2.841A1.5 1.5 0 004 4.11V15.89a1.5 1.5 0 002.3 1.269l9.344-5.89a1.5 1.5 0 000-2.538L6.3 2.84z"/>
              </svg>
              {{ isRecording ? 'Recording...' : 'Create Video' }}
            </span>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.05); }
}

.current-word {
  animation: pulse 2s infinite;
}

.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
