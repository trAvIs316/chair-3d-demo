<template>
  <div class="viewer-container">
    <!-- 2D 懸浮控制面板 -->
    <div class="ui-panel">
      <h2>3D 模型驗證 (Chair Demo)</h2>
      <p>狀態：<span :class="statusColor">{{ loadStatus }}</span></p>
      <div class="btn-group">
        <button @click="resetCamera">重置視角</button>
        <button @click="toggleAutoRotate">{{ isRotating ? '停止旋轉' : '自動旋轉' }}</button>
      </div>
    </div>

    <!-- Three.js 渲染畫布容器 -->
    <div ref="canvasContainer" class="canvas-box"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, computed } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import gsap from 'gsap'

const canvasContainer = ref(null)
const loadStatus = ref('模型載入中...')
const isRotating = ref(true)

const statusColor = computed(() => {
  return loadStatus.value === '載入完成' ? 'status-ready' : 'status-loading'
})

let scene, camera, renderer, controls, reqId, model

const initThree = () => {
  const container = canvasContainer.value
  const width = container.clientWidth
  const height = container.clientHeight

  // 1. 場景
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x1a1a24)

  // 2. 相機
  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 100)
  camera.position.set(2, 2, 3)

  // 3. 渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(width, height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.shadowMap.enabled = true
  container.appendChild(renderer.domElement)

  // 4. 軌道控制器 (支援手機觸控旋轉、雙指縮放)
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.autoRotate = isRotating.value
  controls.autoRotateSpeed = 2.0

  // 5. 燈光配置 (環境光 + 三點布光)
  const ambientLight = new THREE.AmbientLight(0xffffff, 1.2)
  scene.add(ambientLight)

  const dirLight = new THREE.DirectionalLight(0xffffff, 2.0)
  dirLight.position.set(5, 10, 7)
  dirLight.castShadow = true
  scene.add(dirLight)

  const fillLight = new THREE.DirectionalLight(0x58a6ff, 1.0)
  fillLight.position.set(-5, 0, -5)
  scene.add(fillLight)

  // 地面網格輔助線
  const gridHelper = new THREE.GridHelper(10, 20, 0x30363d, 0x21262d)
  scene.add(gridHelper)

  // 6. 載入 public/chair.glb 模型
  // 使用 import.meta.env.BASE_URL 確保在 GitHub Pages 子路徑下能正確讀取
  const loader = new GLTFLoader()
  const modelUrl = `${import.meta.env.BASE_URL}chair.glb`

  loader.load(
    modelUrl,
    (gltf) => {
      model = gltf.scene

      // 自動計算模型包圍盒，將模型置中並貼地
      const box = new THREE.Box3().setFromObject(model)
      const center = box.getCenter(new THREE.Vector3())
      const size = box.getSize(new THREE.Vector3())

      model.position.x -= center.x
      model.position.z -= center.z
      model.position.y -= box.min.y // 讓底座剛好站在網格地面上

      scene.add(model)
      loadStatus.value = '載入完成'

      // 根據模型尺寸自動調校相機視角
      const maxDim = Math.max(size.x, size.y, size.z)
      camera.position.set(maxDim * 1.5, maxDim * 1.2, maxDim * 1.8)
      controls.target.set(0, size.y / 2, 0)
      controls.update()
    },
    (xhr) => {
      if (xhr.total > 0) {
        const percent = Math.round((xhr.loaded / xhr.total) * 100)
        loadStatus.value = `載入中 ${percent}%`
      }
    },
    (error) => {
      console.error('模型載入失敗:', error)
      loadStatus.value = '模型載入失敗，請確認 public/chair.glb 存在'
    }
  )

  // 7. 60 FPS 渲染迴圈
  const animate = () => {
    reqId = requestAnimationFrame(animate)
    controls.update()
    renderer.render(scene, camera)
  }
  animate()
}

// 控制面板功能
const resetCamera = () => {
  gsap.to(camera.position, {
    x: 2,
    y: 2,
    z: 3,
    duration: 1,
    ease: 'power2.inOut',
    onUpdate: () => controls.update()
  })
}

const toggleAutoRotate = () => {
  isRotating.value = !isRotating.value
  controls.autoRotate = isRotating.value
}

// 響應視窗縮放
const onResize = () => {
  if (!canvasContainer.value || !renderer || !camera) return
  const w = canvasContainer.value.clientWidth
  const h = canvasContainer.value.clientHeight
  camera.aspect = w / h
  camera.updateProjectionMatrix()
  renderer.setSize(w, h)
}

onMounted(() => {
  initThree()
  window.addEventListener('resize', onResize)
})

onBeforeUnmount(() => {
  cancelAnimationFrame(reqId)
  window.removeEventListener('resize', onResize)
  renderer?.dispose()
})
</script>

<style scoped>
.viewer-container {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background: #0d1117;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
}

.canvas-box {
  width: 100%;
  height: 100%;
}

.ui-panel {
  position: absolute;
  top: 16px;
  left: 16px;
  z-index: 10;
  background: rgba(22, 27, 34, 0.85);
  border: 1px solid #30363d;
  border-radius: 8px;
  padding: 16px;
  color: #e6edf3;
  backdrop-filter: blur(8px);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.4);
}

h2 {
  font-size: 16px;
  margin: 0 0 8px 0;
  color: #58a6ff;
}

p {
  font-size: 13px;
  margin: 0 0 12px 0;
  color: #8b949e;
}

.status-ready {
  color: #3fb950;
  font-weight: bold;
}

.status-loading {
  color: #f0883e;
}

.btn-group {
  display: flex;
  gap: 8px;
}

button {
  background: #21262d;
  color: #c9d1d9;
  border: 1px solid #30363d;
  padding: 6px 12px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 12px;
  transition: all 0.2s;
}

button:hover {
  background: #30363d;
  color: #fff;
  border-color: #8b949e;
}
</style>