<script setup>
import { ref, onMounted } from 'vue'
import Soundfont from 'soundfont-player'

const drums = ref([
    { id: 'crash', name: 'Crash', sound: 'crash', midiNote: 49 }, // Crash Cymbal 1
    { id: 'tom1', name: 'Tom 1', sound: 'tom1', midiNote: 48 }, // High-Mid Tom
    { id: 'tom2', name: 'Tom 2', sound: 'tom2', midiNote: 45 }, // Low Tom
    { id: 'tom3', name: 'Tom 3', sound: 'tom3', midiNote: 43 }, // High Floor Tom
    { id: 'ride', name: 'Ride', sound: 'ride', midiNote: 51 }, // Ride Cymbal 1
    { id: 'hihat', name: 'Hi-Hat', sound: 'hihat', midiNote: 42 }, // Open Hi-hat
    { id: 'snare', name: 'Snare', sound: 'snare', midiNote: 38 }, // Acoustic Snare
    { id: 'kick', name: 'Kick', sound: 'kick', midiNote: 36 }, // Bass Drum 1
])

const gridSteps = 16
const isPlaying = ref(false)
const drumKit = ref(null)
const tempo = ref(120) // BPM
const audioContext = ref(null)
const isLoading = ref(true)
const errorMessage = ref('')

const pattern = ref(
  drums.value.reduce((acc, drum) => {
    acc[drum.id] = Array(gridSteps).fill(false)
    return acc
  }, {})
)

onMounted(async () => {
  try {
    audioContext.value = new (window.AudioContext || window.webkitAudioContext)()
    
    // Use locally served soundfont
    drumKit.value = await Soundfont.instrument(audioContext.value, 'drums', {
      soundfont: 'FluidR3_GM',
      format: 'mp3',
      nameToUrl: (name, soundfont) => {
        return `/soundfonts/${name}-mp3.js` // Served from public directory
      }
    })

    document.addEventListener('click', () => {
      if (audioContext.value?.state === 'suspended') {
        audioContext.value.resume()
      }
    }, { once: true })

    isLoading.value = false
  } catch (error) {
    console.error('Failed to initialize audio:', error)
    errorMessage.value = 'Failed to load drum sounds. Please refresh the page.'
    isLoading.value = false
  }
})

const toggleCell = (drumId, step) => {
  pattern.value[drumId][step] = !pattern.value[drumId][step]
}

const playPattern = async () => {
  if (!drumKit.value || !audioContext.value) {
    errorMessage.value = 'Drum kit not loaded yet. Please wait or refresh the page.'
    return
  }
  
  isPlaying.value = true
  const stepDuration = (60 / tempo.value) / 4 // Duration of a 16th note in seconds
  
  for (let step = 0; step < gridSteps; step++) {
    if (!isPlaying.value) break
    
    drums.value.forEach(drum => {
      if (pattern.value[drum.id][step]) {
        drumKit.value.play(drum.midiNote, audioContext.value.currentTime, {
          gain: 1.5,
          duration: 0.2
        })
      }
    })
    
    await new Promise(resolve => setTimeout(resolve, stepDuration * 1000))
  }
  
  isPlaying.value = false
}

const stopPattern = () => {
  isPlaying.value = false
}
</script>

<template>
  <div class="max-w-5xl mx-auto">
    <!-- Error message -->
    <div v-if="errorMessage" class="mb-4 p-4 bg-red-100 text-red-700 rounded-md">
      {{ errorMessage }}
    </div>

    <!-- Loading indicator -->
    <div v-if="isLoading" class="mb-4 p-4 bg-blue-100 text-blue-700 rounded-md">
      Loading drum sounds...
    </div>

    <!-- Playback controls -->
    <div class="mb-4 flex gap-2">
      <button
        @click="playPattern"
        :disabled="isPlaying || isLoading"
        class="px-4 py-2 bg-green-500 text-white rounded-md hover:bg-green-600 disabled:bg-gray-400"
      >
        {{ isLoading ? 'Loading...' : 'Play' }}
      </button>
      <button
        @click="stopPattern"
        :disabled="!isPlaying || isLoading"
        class="px-4 py-2 bg-red-500 text-white rounded-md hover:bg-red-600 disabled:bg-gray-400"
      >
        Stop
      </button>
      <div class="flex items-center gap-2 ml-4">
        <label class="text-gray-800 font-medium">Tempo:</label>
        <input
          v-model="tempo"
          type="number"
          min="40"
          max="240"
          class="w-20 px-2 py-1 border border-gray-300 rounded text-gray-800 font-medium focus:outline-none focus:ring-2 focus:ring-green-500 focus:border-transparent"
        />
        <span class="text-gray-800 font-medium">BPM</span>
      </div>
    </div>

    <!-- Step numbers row -->
    <div class="flex mb-2">
      <div class="w-20"></div>
      <div class="flex-1 grid grid-cols-16 gap-1">
        <div v-for="step in gridSteps" :key="step" 
             class="text-center text-sm font-medium text-gray-800">
          {{ step }}
        </div>
      </div>
    </div>

    <!-- Drum rows -->
    <div class="space-y-2">
      <div v-for="drum in drums" :key="drum.id" 
           class="flex items-center">
        <div class="w-20 font-medium text-gray-800">{{ drum.name }}</div>
        <div class="flex-1 grid grid-cols-16 gap-1">
          <div v-for="step in gridSteps" :key="step"
               @click="toggleCell(drum.id, step - 1)"
               class="aspect-square bg-gray-200 rounded cursor-pointer hover:bg-gray-300 border border-gray-300"
               :class="{ 'bg-green-500 hover:bg-green-600 border-green-600': pattern[drum.id][step - 1] }">
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
.grid-cols-16 {
  grid-template-columns: repeat(16, minmax(0, 1fr));
}
</style>