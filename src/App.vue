<script setup>
import { onMounted } from 'vue'
import Soundfont from 'soundfont-player'
import DrumGrid from './components/DrumGrid.vue'  // Add this line

// Initialize audio context
const audioContext = new (window.AudioContext || window.webkitAudioContext)()

// We'll load the drum sounds when the component mounts
onMounted(async () => {
  // Load a basic drum kit
  const drumKit = await Soundfont.instrument(audioContext, 'percussion_kit', {
    soundfont: 'FluidR3_GM'
  })

  // Make sure audio context is running (browsers require user interaction)
  document.addEventListener('click', () => {
    if (audioContext.state === 'suspended') {
      audioContext.resume()
    }
  })
})
</script>

<template>
  <div class="min-h-screen bg-white">
    <div class="container mx-auto p-4">
      <h1 class="text-2xl font-bold mb-4 text-gray-900">Drum Pattern Programmer</h1>
      <DrumGrid />
    </div>
  </div>
</template>

<style>
body {
  margin: 0;
  padding: 0;
}
</style>