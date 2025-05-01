<script setup lang="ts">
import { ref, defineProps, defineEmits } from 'vue'
import { Color3 } from '@babylonjs/core'

const props = defineProps<{
  initialColor: Color3
}>()

const emit = defineEmits(['colorSelected', 'close'])

const selectedColor = ref(props.initialColor.clone())
const hue = ref(0)
const saturation = ref(1)
const value = ref(1)

// Convert HSV to RGB
const hsvToRgb = (h: number, s: number, v: number) => {
  let r = 0, g = 0, b = 0
  
  const i = Math.floor(h * 6)
  const f = h * 6 - i
  const p = v * (1 - s)
  const q = v * (1 - f * s)
  const t = v * (1 - (1 - f) * s)
  
  switch (i % 6) {
    case 0: r = v; g = t; b = p; break
    case 1: r = q; g = v; b = p; break
    case 2: r = p; g = v; b = t; break
    case 3: r = p; g = q; b = v; break
    case 4: r = t; g = p; b = v; break
    case 5: r = v; g = p; b = q; break
  }
  
  return new Color3(r, g, b)
}

// Update color from HSV sliders
const updateColor = () => {
  selectedColor.value = hsvToRgb(hue.value, saturation.value, value.value)
}

// Select color and emit event
const selectColor = () => {
  emit('colorSelected', selectedColor.value)
  emit('close')
}

// Close color picker
const close = () => {
  emit('close')
}
</script>

<template>
  <div class="color-picker-container">
    <div class="color-preview" :style="{ backgroundColor: `rgb(${selectedColor.r * 255}, ${selectedColor.g * 255}, ${selectedColor.b * 255})` }"></div>
    
    <div class="slider-group">
      <label>Hue</label>
      <input type="range" v-model="hue" min="0" max="1" step="0.01" @input="updateColor" />
    </div>
    
    <div class="slider-group">
      <label>Saturation</label>
      <input type="range" v-model="saturation" min="0" max="1" step="0.01" @input="updateColor" />
    </div>
    
    <div class="slider-group">
      <label>Value</label>
      <input type="range" v-model="value" min="0" max="1" step="0.01" @input="updateColor" />
    </div>
    
    <div class="button-group">
      <button @click="selectColor" class="btn primary">Select</button>
      <button @click="close" class="btn">Cancel</button>
    </div>
  </div>
</template>

<style scoped>
.color-picker-container {
  width: 250px;
  padding: 15px;
  background-color: white;
  border-radius: 5px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.2);
}

.color-preview {
  width: 100%;
  height: 50px;
  border-radius: 5px;
  margin-bottom: 15px;
  border: 1px solid #ddd;
}

.slider-group {
  margin-bottom: 10px;
}

.slider-group label {
  display: block;
  margin-bottom: 5px;
  color: #333;
}

.slider-group input {
  width: 100%;
}

.button-group {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 15px;
}

.btn {
  padding: 5px 15px;
  border: none;
  border-radius: 3px;
  cursor: pointer;
  background-color: #e0e0e0;
}

.btn.primary {
  background-color: #3498db;
  color: white;
}

.btn:hover {
  opacity: 0.9;
}
</style>