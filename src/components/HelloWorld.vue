<script setup lang="ts">
import { onMounted, ref } from 'vue'
import { Engine, Scene, ArcRotateCamera, Vector3, HemisphericLight, MeshBuilder, SceneLoader } from '@babylonjs/core'
import '@babylonjs/loaders' // Import side effects for loaders

defineProps<{
  msg: string
}>()
const canvas = ref<HTMLCanvasElement | null>(null)
const createScene = (engine: Engine, canvasElement: HTMLCanvasElement) => {
  const scene = new Scene(engine)

  // Add a camera to the scene and attach it to the canvas
  const camera = new ArcRotateCamera("camera", -Math.PI / 2, Math.PI / 2.5, 10, new Vector3(0, 0, 0), scene)
  camera.attachControl(canvasElement, true)

  // Add a lights to the scene
  const light = new HemisphericLight("light", new Vector3(1, 1, 0), scene)
  light.intensity = 0.7

  // Add a basic sphere
  const sphere = MeshBuilder.CreateSphere("sphere", { diameter: 2, segments: 32 }, scene)
  sphere.position.y = 1

  // Add a ground plane
  const ground = MeshBuilder.CreateGround("ground", { width: 6, height: 6 }, scene)

  // Optional: Load a model (ensure you have a model file, e.g., in public/assets)
  // SceneLoader.ImportMeshAsync("", "/assets/", "yourModel.glb", scene).then((result) => {
  //   // Position or scale the loaded model if needed
  //   // result.meshes[0].position = new Vector3(0, 0, 0);
  // });

  return scene
}
onMounted(() => {
  if (canvas.value) {
    const engine = new Engine(canvas.value, true)
    const scene = createScene(engine, canvas.value)

    engine.runRenderLoop(() => {
      scene.render()
    })

    window.addEventListener('resize', () => {
      engine.resize()
    })
  }
})
</script>

<template>
  <div class="greetings">
    <h1 class="green">{{ msg }}</h1>
    <h3>
      You’ve successfully created a project with
      <a href="https://vitejs.dev/" target="_blank" rel="noopener">Vite</a> +
      <a href="https://vuejs.org/" target="_blank" rel="noopener">Vue 3</a> +
      <a href="https://www.babylonjs.com/" target="_blank" rel="noopener">Babylon.js</a>.
    </h3>
  </div>
  <canvas ref="canvas" style="width: 80vw; height: 70vh; touch-action: none;"></canvas>
</template>

<style scoped>
/* ... existing styles ... */

/* Ensure canvas is visible */
canvas {
  display: block;
  margin-top: 20px;
  border: 1px solid lightgray; /* Optional: Add border to see canvas bounds */
}

/* ... existing styles ... */
</style>