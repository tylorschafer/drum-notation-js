<script setup>
import { ref, onMounted, computed, watch } from 'vue'
import Soundfont from 'soundfont-player'

const drums = ref([
  { id: 'crash', name: 'Crash', sound: 'crash', midiNote: 49 },
  { id: 'ride', name: 'Ride', sound: 'ride', midiNote: 51 },
  { id: 'tom1', name: 'Tom 1', sound: 'tom1', midiNote: 48 },
  { id: 'tom2', name: 'Tom 2', sound: 'tom2', midiNote: 45 },
  { id: 'tom3', name: 'Tom 3', sound: 'tom3', midiNote: 43 },
  { id: 'hihat', name: 'Hi-Hat', sound: 'hihat', midiNote: 42 },
  { id: 'snare', name: 'Snare', sound: 'snare', midiNote: 38 },
  { id: 'kick', name: 'Kick', sound: 'kick', midiNote: 36 }
])

// Pattern structure controls
const barCount = ref(1)
const subdivision = ref(16) // 4 = quarter, 8 = eighth, 16 = sixteenth notes
const timeSignature = ref(4) // 4/4 time signature

// Computed total grid steps
const gridSteps = computed(() => {
  return barCount.value * subdivision.value * (timeSignature.value / 4);
})

// Current playback step
const currentStep = ref(-1)

const isPlaying = ref(false)
const drumKit = ref(null)
const tempo = ref(120)
const audioContext = ref(null)
const isLoading = ref(true)
const errorMessage = ref('')

// Create dynamic grid pattern
const pattern = ref({})

// Initialize or update pattern when grid size changes
const initializePattern = () => {
  const steps = gridSteps.value
  const newPattern = {}
  
  drums.value.forEach(drum => {
    // Copy existing pattern if available, otherwise create new
    newPattern[drum.id] = pattern.value[drum.id] || Array(steps).fill(false)
    
    // If new pattern is larger than old, extend it
    if (newPattern[drum.id].length < steps) {
      newPattern[drum.id] = [...newPattern[drum.id], ...Array(steps - newPattern[drum.id].length).fill(false)]
    }
    // If new pattern is smaller than old, truncate it
    else if (newPattern[drum.id].length > steps) {
      newPattern[drum.id] = newPattern[drum.id].slice(0, steps)
    }
  })
  
  pattern.value = newPattern
}

// Watch for changes that require pattern restructuring
watch([barCount, subdivision, timeSignature], initializePattern)

onMounted(async () => {
  try {
    audioContext.value = new (window.AudioContext || window.webkitAudioContext)()
    
    // Initialize drumKit with locally served soundfont
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

    initializePattern()
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
  const stepDuration = (60 / tempo.value) / (subdivision.value / 4) // Duration of one step
  
  for (let step = 0; step < gridSteps.value; step++) {
    if (!isPlaying.value) break
    
    currentStep.value = step
    
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
  
  currentStep.value = -1
  isPlaying.value = false
}

const stopPattern = () => {
  isPlaying.value = false
  currentStep.value = -1
}

// Helper to check if a step is at a bar line
const isBarStart = (stepIndex) => {
  const stepsPerBar = subdivision.value * (timeSignature.value / 4)
  return stepIndex % stepsPerBar === 0
}

// Helper to check if step is a quarter note
const isQuarterNote = (stepIndex) => {
  const stepsPerQuarter = subdivision.value / 4
  return stepIndex % stepsPerQuarter === 0
}

// Helper for cell CSS classes
const getCellClass = (drumId, step) => {
  // Check if pattern and drumId exist before accessing properties
  if (!pattern.value || !pattern.value[drumId]) {
    return {
      'border-l-2 border-l-gray-500': isBarStart(step), 
      'border-l border-l-gray-300': isQuarterNote(step) && !isBarStart(step)
    }
  }
  
  return {
    'bg-green-500 hover:bg-green-600 border-green-600': pattern.value[drumId][step],
    'border-l-2 border-l-gray-500': isBarStart(step), 
    'border-l border-l-gray-300': isQuarterNote(step) && !isBarStart(step),
    'bg-yellow-200': currentStep.value === step && !pattern.value[drumId][step],
    'bg-yellow-500': currentStep.value === step && pattern.value[drumId][step]
  }
}
</script>

<template>
  <div class="w-full p-0">
    <!-- Error message -->
    <div v-if="errorMessage" class="mb-4 p-4 bg-red-100 text-red-700 rounded-md">
      {{ errorMessage }}
    </div>

    <!-- Loading indicator -->
    <div v-if="isLoading" class="mb-4 p-4 bg-blue-100 text-blue-700 rounded-md">
      Loading drum sounds...
    </div>

    <!-- Pattern controls -->
    <div class="mb-6 p-4 bg-gray-100 rounded-md">
      <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
        <div class="flex flex-col">
          <label class="mb-1 text-gray-700 font-medium">Bars</label>
          <select v-model="barCount" class="px-3 py-2 bg-white border border-gray-300 rounded-md text-gray-800 appearance-none">
            <option :value="1">1 Bar</option>
            <option :value="2">2 Bars</option>
            <option :value="4">4 Bars</option>
            <option :value="8">8 Bars</option>
          </select>
        </div>
        
        <div class="flex flex-col">
          <label class="mb-1 text-gray-700 font-medium">Subdivision</label>
          <select v-model="subdivision" class="px-3 py-2 bg-white border border-gray-300 rounded-md text-gray-800 appearance-none">
            <option :value="4">Quarter Notes</option>
            <option :value="8">Eighth Notes</option>
            <option :value="16">Sixteenth Notes</option>
            <option :value="32">32nd Notes</option>
          </select>
        </div>
        
        <div class="flex flex-col">
          <label class="mb-1 text-gray-700 font-medium">Time Signature</label>
          <select v-model="timeSignature" class="px-3 py-2 bg-white border border-gray-300 rounded-md text-gray-800 appearance-none">
            <option :value="3">3/4</option>
            <option :value="4">4/4</option>
            <option :value="5">5/4</option>
            <option :value="6">6/8</option>
            <option :value="7">7/8</option>
          </select>
        </div>
      </div>
      
      <div class="mt-4 text-sm text-gray-600">
        Grid: {{ gridSteps }} steps ({{ barCount }} bars of {{ subdivision }} steps in {{ timeSignature }}/{{ subdivision === 8 && timeSignature === 6 ? 8 : 4 }} time)
      </div>
    </div>

    <!-- Playback controls -->
    <div class="mb-4 p-4">
      <div class="flex gap-2">
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
    </div>

    <!-- Main grid container -->
    <div class="drum-grid-container">
      <!-- Labels and grid wrapper with shared scrolling -->
      <div class="grid-scroll-wrapper">
        <div class="sticky-labels">
          <div class="empty-corner"></div>
          <div v-for="drum in drums" :key="`label-${drum.id}`" class="instrument-label">
            {{ drum.name }}
          </div>
        </div>
        
        <div class="grid-scroll-container">
          <!-- Beat number labels -->
          <div class="beat-labels-row">
            <template v-for="(_, index) in gridSteps" :key="`beat-${index}`">
              <div 
                class="beat-label" 
                :class="{ 
                  'font-bold': isBarStart(index),
                  'text-gray-900': isBarStart(index),
                  'text-gray-600': !isBarStart(index) && isQuarterNote(index),
                  'text-gray-400': !isQuarterNote(index)
                }"
              >
                {{ isBarStart(index) ? Math.floor(index / (subdivision * (timeSignature / 4))) + 1 : 
                   isQuarterNote(index) ? ((index % (subdivision * (timeSignature / 4))) / (subdivision / 4)) + 1 : '·' }}
              </div>
            </template>
          </div>

          <!-- Drum grid rows -->
          <div class="grid-rows">
            <div v-for="drum in drums" :key="`grid-${drum.id}`" class="drum-row">
              <div 
                v-for="step in gridSteps" 
                :key="`cell-${drum.id}-${step}`"
                @click="toggleCell(drum.id, step - 1)"
                class="grid-cell"
                :class="getCellClass(drum.id, step - 1)"
              ></div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
body, html {
  margin: 0;
  padding: 0;
  width: 100%;
  background-color: white;
}

#app {
  width: 100%;
  background-color: white;
}

.drum-grid-container {
  width: 100%;
  margin-bottom: 2rem;
  position: relative;
  overflow-x: hidden;
}

.grid-scroll-wrapper {
  display: flex;
}

.sticky-labels {
  position: sticky;
  left: 0;
  z-index: 10;
  background-color: white;
  display: flex;
  flex-direction: column;
  width: 80px;
  flex-shrink: 0;
  padding-top: 2px;
}

.empty-corner {
  height: 30px;
  width: 80px;
  flex-shrink: 0;
  margin-bottom: 4px;
}

.instrument-label {
  height: 44px;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  font-weight: 500;
  color: #1f2937;
  padding-right: 1rem;
  background-color: white;
  margin: 2px 0;
}

.grid-scroll-container {
  overflow-x: auto;
  flex-grow: 1;
}

.beat-labels-row {
  display: flex;
  height: 30px;
  align-items: center;
  margin-bottom: 4px;
}

.beat-label {
  width: 40px;
  text-align: center;
  font-size: 0.75rem;
  flex-shrink: 0;
}

.grid-rows {
  display: flex;
  flex-direction: column;
  padding-top: 2px;
}

.drum-row {
  display: flex;
  min-height: 44px;
  align-items: center;
  margin: 2px 0;
}

.grid-cell {
  width: 40px;
  height: 40px;
  margin: 2px;
  flex-shrink: 0;
  background-color: #e5e7eb;
  border: 1px solid #d1d5db;
  border-radius: 0.25rem;
  cursor: pointer;
}

.grid-cell:hover {
  background-color: #d1d5db;
}

.grid-cell.bg-green-500 {
  background-color: #10b981;
  border-color: #059669;
}

.grid-cell.bg-green-500:hover {
  background-color: #059669;
}

.grid-cell.bg-yellow-200 {
  background-color: #fef08a;
}

.grid-cell.bg-yellow-500 {
  background-color: #f59e0b;
}

.grid-cell.border-l-2.border-l-gray-500 {
  border-left: 2px solid #6b7280;
}

.grid-cell.border-l.border-l-gray-300 {
  border-left: 1px solid #d1d5db;
}

/* Fix for dropdown appearance */
select {
  background-image: url("data:image/svg+xml,%3csvg xmlns='http://www.w3.org/2000/svg' fill='none' viewBox='0 0 20 20'%3e%3cpath stroke='%236b7280' stroke-linecap='round' stroke-linejoin='round' stroke-width='1.5' d='M6 8l4 4 4-4'/%3e%3c/svg%3e");
  background-position: right 0.5rem center;
  background-repeat: no-repeat;
  background-size: 1.5em 1.5em;
  padding-right: 2.5rem;
}
</style>