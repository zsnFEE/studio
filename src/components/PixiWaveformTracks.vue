<template>
  <div class="pixi-waveform-tracks">
    <div class="tracks-header">
      <h3>3D 音频轨道</h3>
      <div class="controls">
        <t-button 
          size="small" 
          @click="togglePlayback"
          :theme="isPlaying ? 'danger' : 'primary'"
        >
          {{ isPlaying ? '暂停' : '播放' }}
        </t-button>
        <t-button size="small" @click="resetView">重置视角</t-button>
      </div>
    </div>
    
    <!-- PixiJS Canvas 容器 -->
    <div 
      ref="pixiContainer" 
      class="pixi-canvas-container"
      @mousedown="onMouseDown"
      @mousemove="onMouseMove"
      @mouseup="onMouseUp"
      @wheel="onWheel"
    ></div>
    
    <!-- 轨道信息面板 -->
    <div class="tracks-info">
      <div 
        v-for="track in tracks" 
        :key="track.id"
        class="track-info"
        :class="{ active: track.id === selectedTrack }"
        @click="selectTrack(track.id)"
      >
        <div class="track-icon" :style="{ backgroundColor: track.color }">
          {{ track.type }}
        </div>
        <div class="track-details">
          <span class="track-name">{{ track.name }}</span>
          <span class="track-duration">{{ track.duration }}s</span>
        </div>
        <div class="track-controls">
          <t-button 
            size="small" 
            variant="text"
            :theme="track.isSolo ? 'primary' : 'default'"
            @click.stop="toggleSolo(track.id)"
          >
            S
          </t-button>
          <t-button 
            size="small" 
            variant="text"
            :theme="track.isMuted ? 'danger' : 'default'"
            @click.stop="toggleMute(track.id)"
          >
            M
          </t-button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue'
import * as PIXI from 'pixi.js'

// 响应式数据
const pixiContainer = ref(null)
const selectedTrack = ref(1)
const isPlaying = ref(false)
const isDragging = ref(false)
const lastMousePos = ref({ x: 0, y: 0 })

// PixiJS 相关变量
let app = null
let container3D = null
let trackMeshes = []
let camera = { x: 0, y: 0, z: 5, rotX: 0, rotY: 0 }

// 轨道数据
const tracks = ref([
  {
    id: 1,
    name: '星辰大海',
    type: 'S',
    color: '#e879f9',
    duration: 43,
    waveformData: generateWaveformData(43),
    isSolo: false,
    isMuted: false
  },
  {
    id: 2,
    name: '笛子',
    type: 'S', 
    color: '#10b981',
    duration: 15,
    waveformData: generateWaveformData(15),
    isSolo: false,
    isMuted: false
  },
  {
    id: 3,
    name: '鼓点',
    type: 'R',
    color: '#f59e0b',
    duration: 8,
    waveformData: generateWaveformData(8),
    isSolo: false,
    isMuted: false
  },
  {
    id: 4,
    name: '背景音',
    type: 'A',
    color: '#3b82f6',
    duration: 30,
    waveformData: generateWaveformData(30),
    isSolo: false,
    isMuted: false
  },
  {
    id: 5,
    name: '和声',
    type: 'H',
    color: '#ef4444',
    duration: 25,
    waveformData: generateWaveformData(25),
    isSolo: false,
    isMuted: false
  }
])

// 生成波形数据
function generateWaveformData(duration) {
  const points = duration * 10 // 每秒10个点
  const data = []
  for (let i = 0; i < points; i++) {
    const x = (i / points) * duration
    const amplitude = Math.sin(i * 0.1) * Math.random() * 0.8 + 0.2
    data.push({ x, y: amplitude })
  }
  return data
}

// 初始化 PixiJS
async function initPixi() {
  if (!pixiContainer.value) return

  // 创建 PIXI 应用
  app = new PIXI.Application({
    width: pixiContainer.value.clientWidth,
    height: 400,
    antialias: true,
    backgroundColor: 0x0a0a0a,
    resolution: window.devicePixelRatio || 1,
    autoDensity: true
  })

  pixiContainer.value.appendChild(app.view)

  // 创建3D容器
  container3D = new PIXI.Container()
  app.stage.addChild(container3D)

  // 创建轨道网格
  createTrackMeshes()
  
  // 启动渲染循环
  app.ticker.add(animate)
  
  // 监听窗口大小变化
  window.addEventListener('resize', onResize)
}

// 创建轨道3D网格
function createTrackMeshes() {
  trackMeshes = []
  
  tracks.value.forEach((track, index) => {
    const trackContainer = new PIXI.Container()
    
    // 创建轨道基座
    const trackBase = createTrackBase(track, index)
    trackContainer.addChild(trackBase)
    
    // 创建3D波形
    const waveform3D = createWaveform3D(track, index)
    trackContainer.addChild(waveform3D)
    
    // 创建粒子效果
    const particles = createParticleSystem(track, index)
    trackContainer.addChild(particles)
    
    container3D.addChild(trackContainer)
    trackMeshes.push({
      container: trackContainer,
      base: trackBase,
      waveform: waveform3D,
      particles: particles,
      track: track
    })
  })
}

// 创建轨道基座
function createTrackBase(track, index) {
  const graphics = new PIXI.Graphics()
  const width = 600
  const height = 60
  const y = index * 80 - 160
  
  // 绘制3D效果的轨道基座
  graphics.beginFill(PIXI.utils.hex2rgb(track.color), 0.3)
  graphics.drawRoundedRect(-width/2, y, width, height, 8)
  graphics.endFill()
  
  // 添加光泽效果
  graphics.beginFill(0xffffff, 0.1)
  graphics.drawRoundedRect(-width/2, y, width, 20, 8)
  graphics.endFill()
  
  // 添加边框
  graphics.lineStyle(2, PIXI.utils.hex2rgb(track.color), 0.8)
  graphics.drawRoundedRect(-width/2, y, width, height, 8)
  
  return graphics
}

// 创建3D波形
function createWaveform3D(track, index) {
  const waveformContainer = new PIXI.Container()
  const y = index * 80 - 160
  
  track.waveformData.forEach((point, i) => {
    if (i % 2 === 0) { // 优化性能，每隔一个点绘制
      const x = (point.x / track.duration) * 600 - 300
      const amplitude = point.y * 40
      
      // 创建3D柱状波形
      const bar = new PIXI.Graphics()
      const color = PIXI.utils.hex2rgb(track.color)
      
      // 绘制3D柱子
      bar.beginFill(color, 0.8)
      bar.drawRect(x - 1, y + 30 - amplitude/2, 2, amplitude)
      bar.endFill()
      
      // 添加高光
      bar.beginFill(0xffffff, 0.3)
      bar.drawRect(x - 1, y + 30 - amplitude/2, 1, amplitude * 0.3)
      bar.endFill()
      
      waveformContainer.addChild(bar)
    }
  })
  
  return waveformContainer
}

// 创建粒子系统
function createParticleSystem(track, index) {
  const particleContainer = new PIXI.Container()
  const particleCount = 20
  
  for (let i = 0; i < particleCount; i++) {
    const particle = new PIXI.Graphics()
    particle.beginFill(PIXI.utils.hex2rgb(track.color), 0.6)
    particle.drawCircle(0, 0, Math.random() * 2 + 1)
    particle.endFill()
    
    // 随机位置
    particle.x = (Math.random() - 0.5) * 600
    particle.y = index * 80 - 160 + (Math.random() - 0.5) * 60
    
    // 添加动画属性
    particle.vx = (Math.random() - 0.5) * 2
    particle.vy = (Math.random() - 0.5) * 2
    particle.life = Math.random()
    
    particleContainer.addChild(particle)
  }
  
  return particleContainer
}

// 动画循环
function animate() {
  if (!container3D) return
  
  // 应用3D变换
  apply3DTransform()
  
  // 更新粒子
  updateParticles()
  
  // 播放动画
  if (isPlaying.value) {
    updatePlaybackAnimation()
  }
}

// 应用3D变换
function apply3DTransform() {
  container3D.x = app.screen.width / 2 + camera.x
  container3D.y = app.screen.height / 2 + camera.y
  container3D.scale.set(1 + camera.z * 0.1)
  container3D.rotation = camera.rotY * 0.01
  
  // 应用Y轴旋转的3D效果
  trackMeshes.forEach((mesh, index) => {
    const rotationEffect = Math.sin(camera.rotY * 0.01) * 0.1
    mesh.container.skew.x = rotationEffect
    mesh.container.scale.y = 1 - Math.abs(rotationEffect) * 0.2
  })
}

// 更新粒子
function updateParticles() {
  trackMeshes.forEach((mesh) => {
    if (!mesh.track.isMuted && (mesh.track.isSolo || !tracks.value.some(t => t.isSolo))) {
      mesh.particles.children.forEach((particle) => {
        particle.x += particle.vx
        particle.y += particle.vy
        particle.alpha = particle.life
        particle.life -= 0.02
        
        if (particle.life <= 0) {
          particle.x = (Math.random() - 0.5) * 600
          particle.y = mesh.container.y + (Math.random() - 0.5) * 60
          particle.life = 1
        }
      })
    }
  })
}

// 播放动画
function updatePlaybackAnimation() {
  const time = Date.now() * 0.005
  
  trackMeshes.forEach((mesh, index) => {
    if (!mesh.track.isMuted) {
      // 波形跳动效果
      mesh.waveform.children.forEach((bar, i) => {
        const wave = Math.sin(time + i * 0.1 + index) * 0.1 + 1
        bar.scale.y = wave
      })
      
      // 基座发光效果
      const glowIntensity = Math.sin(time * 2 + index) * 0.2 + 0.8
      mesh.base.alpha = glowIntensity
    }
  })
}

// 鼠标事件处理
function onMouseDown(event) {
  isDragging.value = true
  lastMousePos.value = { x: event.clientX, y: event.clientY }
}

function onMouseMove(event) {
  if (!isDragging.value) return
  
  const deltaX = event.clientX - lastMousePos.value.x
  const deltaY = event.clientY - lastMousePos.value.y
  
  camera.rotY += deltaX * 0.5
  camera.rotX += deltaY * 0.5
  
  lastMousePos.value = { x: event.clientX, y: event.clientY }
}

function onMouseUp() {
  isDragging.value = false
}

function onWheel(event) {
  camera.z += event.deltaY * 0.001
  camera.z = Math.max(-2, Math.min(10, camera.z))
}

// 窗口大小调整
function onResize() {
  if (app && pixiContainer.value) {
    app.renderer.resize(pixiContainer.value.clientWidth, 400)
  }
}

// 控制函数
function selectTrack(trackId) {
  selectedTrack.value = trackId
}

function toggleSolo(trackId) {
  const track = tracks.value.find(t => t.id === trackId)
  if (track) {
    track.isSolo = !track.isSolo
  }
}

function toggleMute(trackId) {
  const track = tracks.value.find(t => t.id === trackId)
  if (track) {
    track.isMuted = !track.isMuted
  }
}

function togglePlayback() {
  isPlaying.value = !isPlaying.value
}

function resetView() {
  camera.x = 0
  camera.y = 0
  camera.z = 5
  camera.rotX = 0
  camera.rotY = 0
}

// 生命周期
onMounted(async () => {
  await nextTick()
  await initPixi()
})

onUnmounted(() => {
  if (app) {
    window.removeEventListener('resize', onResize)
    app.destroy(true)
  }
})
</script>

<style scoped>
.pixi-waveform-tracks {
  width: 100%;
  background: linear-gradient(135deg, #0a0a0a 0%, #1a1a1a 100%);
  border-radius: 12px;
  overflow: hidden;
  border: 1px solid #333;
}

.tracks-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  background: rgba(255, 255, 255, 0.05);
  border-bottom: 1px solid #333;
}

.tracks-header h3 {
  color: #fff;
  margin: 0;
  font-size: 16px;
  font-weight: 600;
}

.controls {
  display: flex;
  gap: 8px;
}

.pixi-canvas-container {
  width: 100%;
  height: 400px;
  position: relative;
  cursor: grab;
}

.pixi-canvas-container:active {
  cursor: grabbing;
}

.tracks-info {
  padding: 16px;
  background: rgba(0, 0, 0, 0.3);
  border-top: 1px solid #333;
}

.track-info {
  display: flex;
  align-items: center;
  padding: 8px 12px;
  margin-bottom: 8px;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
}

.track-info:hover {
  background: rgba(255, 255, 255, 0.1);
  transform: translateX(4px);
}

.track-info.active {
  background: rgba(64, 158, 255, 0.2);
  border: 1px solid #409eff;
}

.track-icon {
  width: 32px;
  height: 32px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  font-weight: bold;
  font-size: 14px;
  margin-right: 12px;
}

.track-details {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.track-name {
  color: #fff;
  font-weight: 500;
  font-size: 14px;
}

.track-duration {
  color: #999;
  font-size: 12px;
}

.track-controls {
  display: flex;
  gap: 4px;
}
</style>