<script setup>
import { ref } from 'vue'

const drums = ref([
    { id: 'tom1', name: 'Tom 1', sound: 'tom1' },
  { id: 'tom2', name: 'Tom 2', sound: 'tom2' },
  { id: 'tom3', name: 'Tom 3', sound: 'tom3' },
  { id: 'crash', name: 'Crash', sound: 'crash' },
  { id: 'ride', name: 'Ride', sound: 'ride' },
  { id: 'hihat', name: 'Hi-Hat', sound: 'hihat' },
  { id: 'snare', name: 'Snare', sound: 'snare' },
  { id: 'kick', name: 'Kick', sound: 'kick' }
])

// Create a 16-step grid
const gridSteps = 16

// Initialize pattern state (all cells off initially)
const pattern = ref(
  drums.value.reduce((acc, drum) => {
    acc[drum.id] = Array(gridSteps).fill(false)
    return acc
  }, {})
)

// Toggle cell state
const toggleCell = (drumId, step) => {
  pattern.value[drumId][step] = !pattern.value[drumId][step]
}
</script>

<template>
  <div class="max-w-5xl mx-auto">
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