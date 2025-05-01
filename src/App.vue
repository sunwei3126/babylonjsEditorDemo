<script setup lang="ts">
import Editor3D from './components/Editor3D.vue'
</script>

<template>
  <div class="app">
    <Editor3D />
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: Arial, sans-serif;
  overflow: hidden;
}

.app {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}
</style>
