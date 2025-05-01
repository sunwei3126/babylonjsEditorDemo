<script setup lang="ts">
import { ref, onMounted, reactive, watch } from 'vue'
import { 
  Engine, Color4, Scene, ArcRotateCamera, Vector3, HemisphericLight, MeshBuilder, 
  Mesh, StandardMaterial, Color3, PointLight, SpotLight, DirectionalLight,
  PBRMaterial, Texture, ActionManager, ExecuteCodeAction, TransformNode,
  AbstractMesh, Tools, GizmoManager
} from '@babylonjs/core'

import { GridMaterial } from '@babylonjs/materials'

// 由于 GridMaterial 未被使用，暂时注释掉这个导入
// Fix the import statement if ColorPickerComponent exists
// import { ColorPickerComponent } from './ColorPicker.vue'
// If it doesn't exist yet, comment it out or remove it

const canvas = ref<HTMLCanvasElement | null>(null)
const colorPickerVisible = ref(false)
const colorPickerPosition = reactive({ x: 0, y: 0 })
// Ensure currentColor is properly initialized with a valid Color3 object
const currentColor = ref(new Color3(1, 1, 1))

// History for undo/redo
const history = reactive({
  past: [] as any[],
  future: [] as any[],
  current: null as any
})

// Scene objects
const sceneState = reactive({
  objects: [] as AbstractMesh[],
  selectedObject: null as AbstractMesh | null,
  transformMode: 'translate', // 'translate', 'rotate', 'scale'
  materials: {
    roughness: 0.5,
    metallic: 0.5,
    normalMap: ''
  }
})

let engine: Engine
let scene: Scene
let gizmoManager: GizmoManager

// Create a scene
const createScene = (engine: Engine, canvasElement: HTMLCanvasElement) => {
  const scene = new Scene(engine)
   scene.clearColor = new Color4(0.05, 0.05, 0.05, 1);

  // Camera
  const camera = new ArcRotateCamera("camera", -Math.PI / 2, Math.PI / 2.5, 10, Vector3.Zero(), scene)
  camera.attachControl(canvasElement, true)
  camera.wheelPrecision = 50
  camera.lowerRadiusLimit = 2
  camera.upperRadiusLimit = 50
  
  // Default lighting
  const hemisphericLight = new HemisphericLight("hemisphericLight", new Vector3(0, 1, 0), scene)
  hemisphericLight.intensity = 0.7
  
  // Ground
  const ground = MeshBuilder.CreateGround("ground", { width: 20, height: 20 }, scene)
  const gridMaterial = new GridMaterial("gridMaterial", scene)
  gridMaterial.majorUnitFrequency = 5;
  gridMaterial.minorUnitVisibility = 0.45;
  gridMaterial.gridRatio = 1;
  gridMaterial.backFaceCulling = false;
  gridMaterial.mainColor = new Color3(1, 1, 1);
  gridMaterial.lineColor = new Color3(1.0, 1.0, 1.0);
  gridMaterial.opacity = 0.98;
  ground.material = gridMaterial;

  // Setup gizmo manager for transformations
  setupGizmoManager(scene)
  
  return scene
}

// Setup gizmo manager for transformations
const setupGizmoManager = (scene: Scene) => {
  // This would be implemented with Babylon's GizmoManager
  // For simplicity, we'll use basic transformation controls in this example
  gizmoManager = new GizmoManager(scene)
  gizmoManager.positionGizmoEnabled = true
  gizmoManager.rotationGizmoEnabled = true
  gizmoManager.scaleGizmoEnabled = true
  gizmoManager.usePointerToAttachGizmos = false 
}

// Add objects
const addCube = () => {
  const cube = MeshBuilder.CreateBox("cube", { size: 1 }, scene)
  cube.position.y = 0.5
  addObjectToScene(cube)
}

const addSphere = () => {
  const sphere = MeshBuilder.CreateSphere("sphere", { diameter: 1 }, scene)
  sphere.position.y = 0.5
  addObjectToScene(sphere)
}

const addCylinder = () => {
  const cylinder = MeshBuilder.CreateCylinder("cylinder", { height: 1, diameter: 1 }, scene)
  cylinder.position.y = 0.5
  addObjectToScene(cylinder)
}

const addPlane = () => {
  const plane = MeshBuilder.CreatePlane("plane", { size: 1 }, scene)
  plane.position.y = 0.5
  addObjectToScene(plane)
}

const addTorus = () => {
  const torus = MeshBuilder.CreateTorus("torus", { diameter: 1, thickness: 0.3 }, scene)
  torus.position.y = 0.5
  addObjectToScene(torus)
}

const addObjectToScene = (mesh: AbstractMesh) => {
  // Add material
  const material = new PBRMaterial(`material_${mesh.name}`, scene)
  material.albedoColor = currentColor.value.clone()
  material.metallic = sceneState.materials.metallic
  material.roughness = sceneState.materials.roughness
  mesh.material = material
  
  // Add to scene objects
  sceneState.objects.push(mesh)
  selectObject(mesh)
  
  // Add to history
  addToHistory({
    type: 'create',
    object: mesh
  })
  
  // Setup click action for selection
  mesh.actionManager = new ActionManager(scene)
  mesh.actionManager.registerAction(
    new ExecuteCodeAction(
      ActionManager.OnPickTrigger,
      () => selectObject(mesh)
    )
  )
}

// Select object
const selectObject = (mesh: AbstractMesh | null) => {
  // Deselect previous
  if (sceneState.selectedObject) {
    // Remove highlight from previous selection
    if (sceneState.selectedObject.material) {
      const material = sceneState.selectedObject.material as PBRMaterial | StandardMaterial
      // Reset any highlight effect if needed
    }
  }
  
  sceneState.selectedObject = mesh
  
  // Attach gizmo to the newly selected object
  if (mesh) {
    gizmoManager.attachToMesh(mesh)
    
    // Update transform mode based on current setting
    updateTransformControls()
    
    // Optional: Add highlight to selected object
    if (mesh.material) {
      const material = mesh.material as PBRMaterial | StandardMaterial
      // Add highlight effect if needed
    }
  } else {
    // If no mesh is selected, detach gizmos
    gizmoManager.attachToMesh(null)
  }
}

// Transform functions
const setTransformMode = (mode: string) => {
  sceneState.transformMode = mode
  updateTransformControls()
}

const updateTransformControls = () => {
  if (!gizmoManager) return
  
  // Disable all gizmos first
  gizmoManager.positionGizmoEnabled = false
  gizmoManager.rotationGizmoEnabled = false
  gizmoManager.scaleGizmoEnabled = false
  
  // Enable the appropriate gizmo based on transform mode
  switch (sceneState.transformMode) {
    case 'translate':
      gizmoManager.positionGizmoEnabled = true
      break
    case 'rotate':
      gizmoManager.rotationGizmoEnabled = true
      break
    case 'scale':
      gizmoManager.scaleGizmoEnabled = true
      break
  }
}

// Material functions
const updateMaterial = () => {
  if (!sceneState.selectedObject || !sceneState.selectedObject.material) return
  
  const material = sceneState.selectedObject.material as PBRMaterial
  material.metallic = sceneState.materials.metallic
  material.roughness = sceneState.materials.roughness
  
  if (sceneState.materials.normalMap) {
    material.bumpTexture = new Texture(sceneState.materials.normalMap, scene)
  }
  
  addToHistory({
    type: 'material',
    object: sceneState.selectedObject,
    properties: { ...sceneState.materials }
  })
}

// Color picker
const openColorPicker = (event: MouseEvent) => {
  colorPickerPosition.x = event.clientX
  colorPickerPosition.y = event.clientY
  colorPickerVisible.value = true
}

const setColor = (color: Color3) => {
  currentColor.value = color
  
  if (sceneState.selectedObject && sceneState.selectedObject.material) {
    const material = sceneState.selectedObject.material as PBRMaterial
    material.albedoColor = color.clone()
    
    addToHistory({
      type: 'color',
      object: sceneState.selectedObject,
      color: color.clone()
    })
  }
}

// Lighting functions
const addPointLight = () => {
  const light = new PointLight("pointLight", new Vector3(0, 2, 0), scene)
  light.intensity = 0.7
  light.diffuse = new Color3(1, 1, 1)
  
  // Add to history
  addToHistory({
    type: 'light',
    light: light
  })
}

const addSpotLight = () => {
  const light = new SpotLight("spotLight", new Vector3(0, 5, 0), new Vector3(0, -1, 0), Math.PI / 4, 2, scene)
  light.intensity = 0.7
  
  // Add to history
  addToHistory({
    type: 'light',
    light: light
  })
}

// History functions
const addToHistory = (action: any) => {
  history.past.push(action)
  history.future = [] // Clear redo stack
  history.current = action
}

const undo = () => {
  if (history.past.length <= 1) return
  
  const current = history.past.pop()
  if (current) {
    history.future.push(current)
    
    // Implement undo logic based on action type
    if (current.type === 'create') {
      const index = sceneState.objects.indexOf(current.object)
      if (index !== -1) {
        sceneState.objects.splice(index, 1)
        current.object.dispose()
      }
    }
    // Implement other undo actions (transform, material, etc.)
  }
}

const redo = () => {
  if (history.future.length === 0) return
  
  const action = history.future.pop()
  if (action) {
    history.past.push(action)
    
    // Implement redo logic based on action type
    // This would recreate objects, apply transformations, etc.
  }
}

const duplicate = () => {
  if (!sceneState.selectedObject) return
  
  // Clone the selected object
  const clone = sceneState.selectedObject.clone(`${sceneState.selectedObject.name}_clone`)
  clone.position.x += 1 // Offset slightly
  
  // Add to scene objects
  addObjectToScene(clone)
}

const deleteObject = () => {
  if (!sceneState.selectedObject) return
  
  const index = sceneState.objects.indexOf(sceneState.selectedObject)
  if (index !== -1) {
    sceneState.objects.splice(index, 1)
    
    addToHistory({
      type: 'delete',
      object: sceneState.selectedObject
    })
    
    sceneState.selectedObject.dispose()
    sceneState.selectedObject = null
  }
}

// Initialize scene
onMounted(() => {
  if (canvas.value) {
    engine = new Engine(canvas.value, true)
    scene = createScene(engine, canvas.value)
    
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
  <n-config-provider :theme="darkTheme">
    <div class="editor-container">
      <!-- Left Sidebar -->
      <div class="sidebar">
        <div class="section">
          <h3>Add Objects</h3>
          <div class="button-group">
            <n-button @click="addCube" type="primary" size="small">Cube</n-button>
            <n-button @click="addSphere" type="primary" size="small">Sphere</n-button>
            <n-button @click="addCylinder" type="primary" size="small">Cylinder</n-button>
          </div>
          <div class="button-group">
            <n-button @click="addPlane" type="primary" size="small">Plane</n-button>
            <n-button @click="addTorus" type="primary" size="small">Torus</n-button>
          </div>
        </div>
        
        <div class="section">
          <h3>Transform</h3>
          <div class="button-group">
            <n-button @click="setTransformMode('translate')" :type="sceneState.transformMode === 'translate' ? 'error' : 'primary'" size="small">Translate</n-button>
            <n-button @click="setTransformMode('rotate')" :type="sceneState.transformMode === 'rotate' ? 'error' : 'primary'" size="small">Rotate</n-button>
            <n-button @click="setTransformMode('scale')" :type="sceneState.transformMode === 'scale' ? 'error' : 'primary'" size="small">Scale</n-button>
          </div>
        </div>
        
        <div class="section">
          <h3>Material</h3>
          <n-select
            v-if="sceneState.selectedObject"
            class="full-width"
            :options="[
              { label: 'Standard', value: 'standard' },
              { label: 'Water', value: 'water' },
              { label: 'Glass', value: 'glass' },
              { label: 'Metal', value: 'metal' }
            ]"
            value="standard"
          />
          
          <div class="color-preview" @click="openColorPicker">
            <div class="color-box" :style="{ backgroundColor: currentColor.value ? `rgb(${currentColor.value.r * 255}, ${currentColor.value.g * 255}, ${currentColor.value.b * 255})` : '#ffffff' }"></div>
          </div>
          
          <div class="slider-group">
            <label>Roughness</label>
            <n-slider v-model:value="sceneState.materials.roughness" :min="0" :max="1" :step="0.01" @update:value="updateMaterial" />
          </div>
          
          <div class="slider-group">
            <label>Metallic</label>
            <n-slider v-model:value="sceneState.materials.metallic" :min="0" :max="1" :step="0.01" @update:value="updateMaterial" />
          </div>
        </div>
        
        <div class="section">
          <h3>Advanced Material</h3>
          <select class="full-width">
            <option value="normal">Normal Map</option>
            <option value="bump">Bump Map</option>
            <option value="specular">Specular Map</option>
          </select>
          <button class="btn apply-btn">Apply</button>
        </div>
        
        <div class="section">
          <h3>Transform</h3>
          <div class="button-group">
            <button class="btn">Translate</button>
            <button class="btn">Rotate</button>
            <button class="btn">Scale</button>
          </div>
          <button class="btn">Local Transform</button>
        </div>
        
        <div class="section">
          <h3>Lighting</h3>
          <div class="button-group">
            <button @click="addPointLight" class="btn">Point Light</button>
            <button @click="addSpotLight" class="btn">Spot Light</button>
          </div>
        </div>
      </div>
      
      <!-- Main Canvas Area -->
      <div class="canvas-container">
        <!-- Top Toolbar -->
        <div class="toolbar">
          <n-button @click="undo" size="small">Undo</n-button>
          <n-button @click="redo" size="small">Redo</n-button>
          <n-button @click="duplicate" size="small">Duplicate</n-button>
          <n-button @click="deleteObject" size="small">Delete</n-button>
          <n-button size="small">📷</n-button>
        </div>
        
        <!-- Canvas -->
        <canvas ref="canvas"></canvas>
        
        <!-- Object List (can be toggled) -->
        <div class="object-list">
          <div v-for="obj in sceneState.objects" :key="obj.id" 
               @click="selectObject(obj)"
               :class="{ selected: sceneState.selectedObject === obj }">
            {{ obj.name }}
          </div>
        </div>
        
        <!-- Properties Panel (when object selected) -->
        <div v-if="sceneState.selectedObject" class="properties-panel">
          <h3>Properties</h3>
          <div class="property">
            <label>Position X</label>
            <input type="number" v-model="sceneState.selectedObject.position.x" />
          </div>
          <div class="property">
            <label>Position Y</label>
            <input type="number" v-model="sceneState.selectedObject.position.y" />
          </div>
          <div class="property">
            <label>Position Z</label>
            <input type="number" v-model="sceneState.selectedObject.position.z" />
          </div>
        </div>
        
        <!-- Color Picker (Popup) -->
        <div v-if="colorPickerVisible" class="color-picker" :style="{ left: colorPickerPosition.x + 'px', top: colorPickerPosition.y + 'px' }">
          <!-- Simple color picker implementation -->
          <div class="color-wheel">
            <!-- Color wheel would be implemented here -->
          </div>
          <div class="color-controls">
            <button @click="colorPickerVisible = false">Close</button>
          </div>
        </div>
      </div>
    </div>
  </n-config-provider>
</template>

<style scoped>
.editor-container {
  display: flex;
  height: 100vh;
  width: 100%;
  overflow: hidden;
}

.sidebar {
  width: 220px;
  background-color: #2c3e50;
  color: white;
  padding: 10px;
  overflow-y: auto;
}

.canvas-container {
  flex: 1;
  display: flex;
  flex-direction: column;
  position: relative;
}

.toolbar {
  background-color: #34495e;
  padding: 5px;
  display: flex;
  gap: 5px;
}

canvas {
  flex: 1;
  width: 100%;
  height: 100%;
  display: block;
}

.section {
  margin-bottom: 15px;
  border-bottom: 1px solid #3d566e;
  padding-bottom: 10px;
}

.section h3 {
  margin-top: 0;
  margin-bottom: 10px;
  font-size: 16px;
}

.button-group {
  display: flex;
  gap: 5px;
  margin-bottom: 5px;
}

.btn {
  background-color: #3498db;
  color: white;
  border: none;
  padding: 5px 10px;
  border-radius: 3px;
  cursor: pointer;
}

.btn:hover {
  background-color: #2980b9;
}

.btn.active {
  background-color: #e74c3c;
}

.full-width {
  width: 100%;
  margin-bottom: 10px;
}

.color-preview {
  margin: 10px 0;
  cursor: pointer;
}

.color-box {
  width: 100%;
  height: 30px;
  border-radius: 3px;
  border: 1px solid #ddd;
}

.slider-group {
  margin-bottom: 10px;
}

.slider-group label {
  display: block;
  margin-bottom: 5px;
}

.apply-btn {
  width: 100%;
  margin-top: 5px;
}

.object-list {
  position: absolute;
  left: 10px;
  top: 50px;
  background-color: rgba(44, 62, 80, 0.8);
  color: white;
  padding: 10px;
  border-radius: 5px;
  max-height: 300px;
  overflow-y: auto;
  width: 150px;
}

.object-list div {
  padding: 5px;
  cursor: pointer;
}

.object-list div.selected {
  background-color: #3498db;
}

.properties-panel {
  position: absolute;
  right: 10px;
  bottom: 10px;
  background-color: rgba(44, 62, 80, 0.8);
  color: white;
  padding: 10px;
  border-radius: 5px;
  width: 200px;
}

.property {
  margin-bottom: 10px;
}

.property label {
  display: block;
  margin-bottom: 5px;
}

.property input {
  width: 100%;
}

.color-picker {
  position: fixed;
  background-color: white;
  border: 1px solid #ddd;
  padding: 10px;
  border-radius: 5px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.2);
  z-index: 1000;
}

.color-wheel {
  width: 200px;
  height: 200px;
  background: conic-gradient(
    red, yellow, lime, aqua, blue, magenta, red
  );
  border-radius: 50%;
  margin-bottom: 10px;
}

.color-controls {
  display: flex;
  justify-content: flex-end;
}
</style>