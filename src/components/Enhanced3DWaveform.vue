<template>
  <div class="enhanced-3d-waveform">
    <div class="controls-panel">
      <div class="panel-left">
        <h3>🎵 3D 音频混音台</h3>
        <div class="view-controls">
          <t-button size="small" @click="setView('front')">正视图</t-button>
          <t-button size="small" @click="setView('side')">侧视图</t-button>
          <t-button size="small" @click="setView('top')">俯视图</t-button>
          <t-button size="small" @click="setView('free')">自由视角</t-button>
        </div>
      </div>
      
      <div class="panel-center">
        <div class="master-controls">
          <t-button 
            size="large"
            :theme="globalPlaying ? 'danger' : 'primary'"
            @click="toggleGlobalPlayback"
          >
            {{ globalPlaying ? '⏸️ 暂停' : '▶️ 播放' }}
          </t-button>
          
          <div class="master-volume">
            <span>主音量</span>
            <t-slider 
              v-model="masterVolume" 
              :min="0" 
              :max="100"
              @change="updateMasterVolume"
            />
          </div>
        </div>
      </div>
      
      <div class="panel-right">
        <div class="effect-controls">
          <t-switch v-model="effects.particles" label="粒子效果" />
          <t-switch v-model="effects.glow" label="发光效果" />
          <t-switch v-model="effects.reflection" label="反射效果" />
          <t-switch v-model="effects.bloom" label="光晕效果" />
        </div>
      </div>
    </div>

    <!-- 3D Canvas 区域 -->
    <div 
      ref="canvasContainer" 
      class="canvas-3d-container"
      @mousedown="handleMouseDown"
      @mousemove="handleMouseMove"
      @mouseup="handleMouseUp"
      @wheel="handleWheel"
      @contextmenu.prevent
    >
      <div class="loading-indicator" v-if="isLoading">
        <t-loading size="large" text="加载3D引擎中..." />
      </div>
    </div>

    <!-- 轨道控制面板 -->
    <div class="tracks-control-panel">
      <div 
        v-for="track in tracks" 
        :key="track.id"
        class="track-controller"
        :class="{ 
          active: track.id === selectedTrack,
          muted: track.isMuted,
          solo: track.isSolo
        }"
      >
        <div class="track-header">
          <div class="track-icon" :style="{ background: track.color }">
            {{ track.type }}
          </div>
          <div class="track-info">
            <h4>{{ track.name }}</h4>
            <span class="track-stats">{{ track.duration }}s | {{ track.sampleRate }}Hz</span>
          </div>
        </div>
        
        <div class="track-controls">
          <div class="volume-control">
            <t-slider 
              v-model="track.volume" 
              :min="0" 
              :max="100"
              size="small"
              @change="updateTrackVolume(track.id, $event)"
            />
            <span class="volume-label">{{ track.volume }}%</span>
          </div>
          
          <div class="track-buttons">
            <t-button 
              size="small"
              :theme="track.isSolo ? 'warning' : 'default'"
              @click="toggleSolo(track.id)"
            >
              SOLO
            </t-button>
            <t-button 
              size="small"
              :theme="track.isMuted ? 'danger' : 'default'"
              @click="toggleMute(track.id)"
            >
              MUTE
            </t-button>
            <t-button 
              size="small"
              theme="primary"
              @click="selectTrack(track.id)"
            >
              SELECT
            </t-button>
          </div>
        </div>
        
        <div class="track-effects">
          <div class="eq-visualizer">
            <div 
              v-for="(band, index) in track.eqBands" 
              :key="index"
              class="eq-band"
              :style="{ 
                height: band.value + '%',
                backgroundColor: track.color,
                opacity: 0.7 + band.value * 0.003
              }"
            ></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick, computed } from 'vue'
import * as PIXI from 'pixi.js'

// 响应式状态
const canvasContainer = ref(null)
const isLoading = ref(true)
const selectedTrack = ref(1)
const globalPlaying = ref(false)
const masterVolume = ref(75)
const currentView = ref('free')

// 3D相机控制
const camera = ref({
  x: 0,
  y: 0, 
  z: 8,
  rotX: 0,
  rotY: 0,
  rotZ: 0,
  fov: 60
})

// 鼠标控制
const mouse = ref({
  isDown: false,
  lastX: 0,
  lastY: 0,
  sensitivity: 0.5
})

// 效果开关
const effects = ref({
  particles: true,
  glow: true,
  reflection: true,
  bloom: true
})

// PixiJS 实例
let app = null
let stage3D = null
let trackMeshes = []
let particleSystems = []
let reflectionLayer = null

// 轨道数据
const tracks = ref([
  {
    id: 1,
    name: '星辰大海',
    type: 'LEAD',
    color: '#e879f9',
    duration: 43,
    volume: 80,
    sampleRate: 44100,
    waveformData: [],
    eqBands: [],
    isSolo: false,
    isMuted: false,
    isPlaying: false
  },
  {
    id: 2,
    name: '空灵笛声',
    type: 'WIND',
    color: '#10b981',
    duration: 28,
    volume: 65,
    sampleRate: 48000,
    waveformData: [],
    eqBands: [],
    isSolo: false,
    isMuted: false,
    isPlaying: false
  },
  {
    id: 3,
    name: '雷鸣鼓点',
    type: 'PERC',
    color: '#f59e0b',
    duration: 35,
    volume: 90,
    sampleRate: 44100,
    waveformData: [],
    eqBands: [],
    isSolo: false,
    isMuted: false,
    isPlaying: false
  },
  {
    id: 4,
    name: '深邃贝斯',
    type: 'BASS',
    color: '#3b82f6',
    duration: 40,
    volume: 70,
    sampleRate: 44100,
    waveformData: [],
    eqBands: [],
    isSolo: false,
    isMuted: false,
    isPlaying: false
  },
  {
    id: 5,
    name: '天籁和声',
    type: 'PAD',
    color: '#ef4444',
    duration: 50,
    volume: 55,
    sampleRate: 96000,
    waveformData: [],
    eqBands: [],
    isSolo: false,
    isMuted: false,
    isPlaying: false
  }
])

// 生成真实音频波形数据
function generateAdvancedWaveform(track) {
  const pointsPerSecond = 100 // 增加密度
  const totalPoints = track.duration * pointsPerSecond
  const waveform = []
  
  for (let i = 0; i < totalPoints; i++) {
    const time = i / pointsPerSecond
    const progress = i / totalPoints
    
    let amplitude = 0
    
    // 根据轨道类型生成不同的波形特征
    switch(track.type) {
      case 'LEAD': // 主旋律 - 清晰的正弦波加谐波
        amplitude = Math.sin(time * 4) * 0.8
        amplitude += Math.sin(time * 8) * 0.3
        amplitude += Math.sin(time * 16) * 0.1
        amplitude *= Math.sin(progress * Math.PI) // 包络线
        break
        
      case 'BASS': // 低音 - 方波特征
        amplitude = Math.sign(Math.sin(time * 2)) * 0.9
        amplitude += Math.sin(time * 4) * 0.2
        amplitude *= (0.8 + Math.sin(time * 0.5) * 0.2) // 低频调制
        break
        
      case 'PERC': // 打击乐 - 脉冲和瞬态
        const beat = Math.floor(time * 4) % 4
        if (beat === 0) {
          amplitude = Math.exp(-((time % 1) * 8)) * Math.sin(time * 40) * 1.2
        } else if (beat === 2) {
          amplitude = Math.exp(-((time % 1) * 6)) * Math.sin(time * 30) * 0.8
        } else {
          amplitude = Math.exp(-((time % 1) * 12)) * Math.sin(time * 60) * 0.4
        }
        break
        
      case 'WIND': // 管乐 - 平滑的正弦波
        amplitude = Math.sin(time * 3 + Math.sin(time * 0.5)) * 0.7
        amplitude += Math.sin(time * 6) * 0.2
        amplitude *= (0.9 + Math.sin(time * 0.3) * 0.1) // 轻微颤音
        break
        
      case 'PAD': // 垫音 - 厚重的和弦
        amplitude = Math.sin(time * 2) * 0.4
        amplitude += Math.sin(time * 2.5) * 0.3
        amplitude += Math.sin(time * 3) * 0.2
        amplitude += Math.sin(time * 4) * 0.1
        amplitude *= 0.8
        break
        
      default:
        amplitude = Math.sin(time * 4) * 0.6
    }
    
    // 添加细微的随机变化模拟真实音频
    amplitude += (Math.random() - 0.5) * 0.05
    
    // 归一化到 [0, 1] 范围
    amplitude = Math.max(0, Math.min(1, Math.abs(amplitude)))
    
    waveform.push({
      time,
      amplitude,
      left: amplitude * (0.9 + Math.random() * 0.2), // 立体声左声道
      right: amplitude * (0.9 + Math.random() * 0.2) // 立体声右声道
    })
  }
  
  return waveform
}

// 生成EQ频段数据
function generateEQBands(track) {
  const bands = []
  const bandCount = 16
  
  for (let i = 0; i < bandCount; i++) {
    const frequency = 20 * Math.pow(2, (i / bandCount) * 10) // 20Hz to 20kHz
    let value = Math.random() * 50 + 10
    
    // 根据轨道类型调整EQ
    switch(track.type) {
      case 'BASS':
        if (i < 4) value *= 2 // 增强低频
        break
      case 'LEAD':
        if (i > 8 && i < 12) value *= 1.5 // 增强中高频
        break
      case 'PERC':
        if (i > 6 && i < 10) value *= 1.8 // 增强中频
        break
    }
    
    bands.push({ frequency, value: Math.min(80, value) })
  }
  
  return bands
}

// 初始化3D场景
async function init3DScene() {
  if (!canvasContainer.value) return
  
  try {
    // 创建PixiJS应用
    app = new PIXI.Application({
      width: canvasContainer.value.clientWidth,
      height: 500,
      antialias: true,
      backgroundColor: 0x000000,
      resolution: window.devicePixelRatio || 1,
      autoDensity: true
    })
    
    canvasContainer.value.appendChild(app.view)
    
    // 创建3D舞台
    stage3D = new PIXI.Container()
    app.stage.addChild(stage3D)
    
    // 生成轨道数据
    tracks.value.forEach(track => {
      track.waveformData = generateAdvancedWaveform(track)
      track.eqBands = generateEQBands(track)
    })
    
    // 创建反射层
    if (effects.value.reflection) {
      createReflectionLayer()
    }
    
    // 创建3D轨道
    await create3DTracks()
    
    // 创建粒子系统
    if (effects.value.particles) {
      createParticleSystems()
    }
    
    // 启动渲染循环
    app.ticker.add(renderLoop)
    
    // 绑定窗口事件
    window.addEventListener('resize', handleResize)
    
    isLoading.value = false
    
  } catch (error) {
    console.error('3D场景初始化失败:', error)
    isLoading.value = false
  }
}

// 创建3D轨道
async function create3DTracks() {
  trackMeshes = []
  
  for (let i = 0; i < tracks.value.length; i++) {
    const track = tracks.value[i]
    const trackContainer = new PIXI.Container()
    
    // 计算轨道位置
    const yPos = (i - tracks.value.length / 2) * 120
    
    // 创建轨道基础
    const trackBase = createTrack3DBase(track, yPos)
    trackContainer.addChild(trackBase)
    
    // 创建3D波形网格
    const waveformMesh = create3DWaveformMesh(track, yPos)
    trackContainer.addChild(waveformMesh)
    
    // 创建频谱分析器
    const spectrumAnalyzer = createSpectrumAnalyzer(track, yPos)
    trackContainer.addChild(spectrumAnalyzer)
    
    // 添加发光效果
    if (effects.value.glow) {
      const glowEffect = createGlowEffect(track, yPos)
      trackContainer.addChild(glowEffect)
    }
    
    stage3D.addChild(trackContainer)
    
    trackMeshes.push({
      container: trackContainer,
      base: trackBase,
      waveform: waveformMesh,
      spectrum: spectrumAnalyzer,
      track: track,
      yPosition: yPos
    })
  }
}

// 创建3D轨道基座
function createTrack3DBase(track, yPos) {
  const graphics = new PIXI.Graphics()
  const width = 800
  const height = 80
  const depth = 20
  
  // 主体颜色
  const color = PIXI.utils.hex2rgb(track.color)
  
  // 绘制3D基座
  graphics.beginFill(color, 0.4)
  graphics.drawRoundedRect(-width/2, yPos - height/2, width, height, 12)
  graphics.endFill()
  
  // 绘制3D边缘效果
  graphics.beginFill(color, 0.6)
  graphics.drawRoundedRect(-width/2, yPos - height/2, width, 8, 12) // 顶部高光
  graphics.drawRoundedRect(-width/2, yPos + height/2 - 8, width, 8, 12) // 底部阴影
  graphics.endFill()
  
  // 绘制侧面
  graphics.beginFill(color, 0.3)
  graphics.moveTo(-width/2 + depth, yPos - height/2 + depth)
  graphics.lineTo(width/2 + depth, yPos - height/2 + depth)
  graphics.lineTo(width/2 + depth, yPos + height/2 + depth)
  graphics.lineTo(-width/2 + depth, yPos + height/2 + depth)
  graphics.closePath()
  graphics.endFill()
  
  return graphics
}

// 创建3D波形网格
function create3DWaveformMesh(track, yPos) {
  const waveformContainer = new PIXI.Container()
  const waveformData = track.waveformData
  const width = 800
  const resolution = 2 // 提高分辨率，每2个像素一个顶点
  const maxAmplitude = 50 // 最大波形高度
  const baselineY = yPos // 波形基线位置
  
  // 创建立体声波形（上下分别显示左右声道）
  for (let i = 0; i < waveformData.length; i += resolution) {
    const point = waveformData[i]
    const nextPoint = waveformData[i + resolution] || point
    const x = (i / waveformData.length) * width - width/2
    const nextX = ((i + resolution) / waveformData.length) * width - width/2
    
    // 左声道（上半部分）
    const leftAmplitude = point.left * maxAmplitude
    const nextLeftAmplitude = nextPoint.left * maxAmplitude
    
    // 右声道（下半部分）  
    const rightAmplitude = point.right * maxAmplitude
    const nextRightAmplitude = nextPoint.right * maxAmplitude
    
    // 创建波形条
    const waveBar = new PIXI.Graphics()
    const color = PIXI.utils.hex2rgb(track.color)
    
    // 绘制左声道波形（向上）
    waveBar.beginFill(color, 0.85)
    waveBar.moveTo(x, baselineY)
    waveBar.lineTo(x, baselineY - leftAmplitude)
    waveBar.lineTo(nextX, baselineY - nextLeftAmplitude)
    waveBar.lineTo(nextX, baselineY)
    waveBar.closePath()
    waveBar.endFill()
    
    // 绘制右声道波形（向下）
    waveBar.beginFill(color, 0.75)
    waveBar.moveTo(x, baselineY)
    waveBar.lineTo(x, baselineY + rightAmplitude)
    waveBar.lineTo(nextX, baselineY + nextRightAmplitude)
    waveBar.lineTo(nextX, baselineY)
    waveBar.closePath()
    waveBar.endFill()
    
    // 添加中心线
    if (i % (resolution * 4) === 0) {
      waveBar.lineStyle(1, color, 0.3)
      waveBar.moveTo(x, baselineY - maxAmplitude * 0.1)
      waveBar.lineTo(x, baselineY + maxAmplitude * 0.1)
    }
    
    // 添加高光效果
    const glowIntensity = Math.max(leftAmplitude, rightAmplitude) / maxAmplitude
    if (glowIntensity > 0.5) {
      waveBar.beginFill(0xffffff, (glowIntensity - 0.5) * 0.4)
      waveBar.moveTo(x, baselineY)
      waveBar.lineTo(x, baselineY - leftAmplitude * 0.3)
      waveBar.lineTo(nextX, baselineY - nextLeftAmplitude * 0.3)
      waveBar.lineTo(nextX, baselineY)
      waveBar.closePath()
      waveBar.endFill()
    }
    
    waveformContainer.addChild(waveBar)
  }
  
  // 添加波形边框
  const border = new PIXI.Graphics()
  border.lineStyle(2, PIXI.utils.hex2rgb(track.color), 0.6)
  border.drawRoundedRect(-width/2 - 5, baselineY - maxAmplitude - 5, width + 10, maxAmplitude * 2 + 10, 4)
  waveformContainer.addChild(border)
  
  return waveformContainer
}

// 创建频谱分析器
function createSpectrumAnalyzer(track, yPos) {
  const spectrumContainer = new PIXI.Container()
  const eqBands = track.eqBands
  const width = 800
  const bandWidth = width / eqBands.length
  const maxHeight = 25 // 降低高度避免重叠
  const offsetY = 65 // 向下偏移更多避免与波形重叠
  
  eqBands.forEach((band, index) => {
    const x = index * bandWidth - width/2
    const height = (band.value / 80) * maxHeight // 归一化高度
    
    const barGraphics = new PIXI.Graphics()
    const color = PIXI.utils.hex2rgb(track.color)
    
    // EQ柱状图 - 渐变效果
    const gradient = new PIXI.Graphics()
    
    // 主体
    gradient.beginFill(color, 0.4)
    gradient.drawRect(x, yPos + offsetY, bandWidth - 2, height)
    gradient.endFill()
    
    // 顶部高亮
    gradient.beginFill(color, 0.8)
    gradient.drawRect(x, yPos + offsetY, bandWidth - 2, Math.min(height, 3))
    gradient.endFill()
    
    // 添加频率标签（仅在特定位置）
    if (index % 4 === 0) {
      const freqLabel = new PIXI.Graphics()
      freqLabel.lineStyle(1, color, 0.3)
      freqLabel.moveTo(x + (bandWidth - 2) / 2, yPos + offsetY)
      freqLabel.lineTo(x + (bandWidth - 2) / 2, yPos + offsetY + maxHeight + 5)
      gradient.addChild(freqLabel)
    }
    
    spectrumContainer.addChild(gradient)
  })
  
  // 添加频谱背景框
  const spectrumBg = new PIXI.Graphics()
  spectrumBg.beginFill(0x000000, 0.2)
  spectrumBg.lineStyle(1, PIXI.utils.hex2rgb(track.color), 0.3)
  spectrumBg.drawRoundedRect(-width/2 - 2, yPos + offsetY - 2, width + 4, maxHeight + 4, 4)
  spectrumBg.endFill()
  spectrumContainer.addChildAt(spectrumBg, 0)
  
  return spectrumContainer
}

// 创建发光效果
function createGlowEffect(track, yPos) {
  const glowContainer = new PIXI.Container()
  
  // 创建发光光圈
  const glow = new PIXI.Graphics()
  const color = PIXI.utils.hex2rgb(track.color)
  
  glow.beginFill(color, 0.1)
  glow.drawEllipse(0, yPos, 850, 100)
  glow.endFill()
  
  glow.beginFill(color, 0.05)
  glow.drawEllipse(0, yPos, 900, 120)
  glow.endFill()
  
  glowContainer.addChild(glow)
  return glowContainer
}

// 创建反射层
function createReflectionLayer() {
  reflectionLayer = new PIXI.Container()
  reflectionLayer.alpha = 0.3
  reflectionLayer.scale.y = -0.5
  reflectionLayer.y = 300
  stage3D.addChild(reflectionLayer)
}

// 创建粒子系统
function createParticleSystems() {
  particleSystems = []
  
  tracks.value.forEach((track, index) => {
    const particleSystem = createTrackParticles(track, index)
    stage3D.addChild(particleSystem.container)
    particleSystems.push(particleSystem)
  })
}

function createTrackParticles(track, trackIndex) {
  const container = new PIXI.Container()
  const particles = []
  const particleCount = 50
  
  for (let i = 0; i < particleCount; i++) {
    const particle = new PIXI.Graphics()
    const size = Math.random() * 3 + 1
    
    particle.beginFill(PIXI.utils.hex2rgb(track.color), 0.7)
    particle.drawCircle(0, 0, size)
    particle.endFill()
    
    // 粒子属性
    particle.x = (Math.random() - 0.5) * 800
    particle.y = (trackIndex - tracks.value.length / 2) * 120
    particle.vx = (Math.random() - 0.5) * 2
    particle.vy = (Math.random() - 0.5) * 2
    particle.life = Math.random()
    particle.maxLife = particle.life
    
    container.addChild(particle)
    particles.push(particle)
  }
  
  return { container, particles, track }
}

// 渲染循环
function renderLoop() {
  if (!stage3D) return
  
  // 应用3D变换
  apply3DTransforms()
  
  // 更新轨道动画
  updateTrackAnimations()
  
  // 更新粒子系统
  if (effects.value.particles) {
    updateParticleSystems()
  }
  
  // 更新EQ可视化
  updateEQVisualization()
}

// 应用3D变换
function apply3DTransforms() {
  const cam = camera.value
  
  // 设置舞台位置和缩放
  stage3D.x = app.screen.width / 2 + cam.x
  stage3D.y = app.screen.height / 2 + cam.y
  
  const scale = 1 + cam.z * 0.05
  stage3D.scale.set(scale)
  
  // 应用旋转
  stage3D.rotation = cam.rotZ * 0.01
  
  // 应用3D透视效果
  trackMeshes.forEach((mesh, index) => {
    const rotEffect = Math.sin(cam.rotY * 0.005) * 0.3
    mesh.container.skew.x = rotEffect
    mesh.container.scale.y = 1 - Math.abs(rotEffect) * 0.1
    
    // 深度排序
    const depth = Math.cos(cam.rotY * 0.005 + index * 0.5)
    mesh.container.zIndex = depth
  })
  
  // 更新反射层
  if (reflectionLayer) {
    reflectionLayer.x = stage3D.x
    reflectionLayer.y = app.screen.height - stage3D.y + 200
    reflectionLayer.scale.x = stage3D.scale.x
    reflectionLayer.scale.y = -stage3D.scale.y * 0.5
  }
}

// 更新轨道动画
function updateTrackAnimations() {
  const time = Date.now() * 0.001
  
  trackMeshes.forEach((mesh, index) => {
    const track = mesh.track
    
    if (globalPlaying.value && !track.isMuted) {
      // 波形动态效果
      if (mesh.waveform && mesh.waveform.children) {
        mesh.waveform.children.forEach((waveBar, i) => {
          if (waveBar.children) return // 跳过边框元素
          
          // 根据音轨类型调整动画频率
          let animationSpeed = 4
          switch(track.type) {
            case 'PERC':
              animationSpeed = 8 // 打击乐更快节奏
              break
            case 'BASS':
              animationSpeed = 2 // 低音更慢节奏
              break
            case 'LEAD':
              animationSpeed = 6 // 主旋律中等速度
              break
            default:
              animationSpeed = 4
          }
          
          const wave = Math.sin(time * animationSpeed + i * 0.05 + index) * 0.1 + 1
          const intensity = Math.sin(time * 2 + index) * 0.2 + 0.9
          
          // 应用缩放和透明度动画
          waveBar.scale.set(1, wave)
          waveBar.alpha = intensity
          
          // 添加轻微的位置抖动（模拟音频振动）
          waveBar.y = Math.sin(time * 10 + i * 0.2) * 0.5
        })
      }
      
      // 基座脉冲效果
      const pulse = Math.sin(time * 2 + index * 0.7) * 0.15 + 1
      mesh.base.scale.y = pulse
      mesh.base.alpha = 0.6 + pulse * 0.4
      
      // 添加轻微的旋转效果
      mesh.container.rotation = Math.sin(time * 0.5 + index) * 0.02
    } else {
      // 静止状态时逐渐恢复原位
      if (mesh.waveform && mesh.waveform.children) {
        mesh.waveform.children.forEach((waveBar) => {
          if (waveBar.children) return
          waveBar.scale.set(1, 1)
          waveBar.alpha = 0.7
          waveBar.y = 0
        })
      }
      mesh.base.scale.y = 1
      mesh.base.alpha = 0.5
      mesh.container.rotation = 0
    }
  })
}

// 更新粒子系统
function updateParticleSystems() {
  particleSystems.forEach((system) => {
    if (!system.track.isMuted && globalPlaying.value) {
      system.particles.forEach((particle) => {
        // 更新位置
        particle.x += particle.vx
        particle.y += particle.vy
        
        // 更新生命周期
        particle.life -= 0.01
        particle.alpha = particle.life / particle.maxLife
        
        // 重置粒子
        if (particle.life <= 0) {
          particle.x = (Math.random() - 0.5) * 800
          particle.y = (Math.random() - 0.5) * 600
          particle.life = particle.maxLife
          particle.vx = (Math.random() - 0.5) * 2
          particle.vy = (Math.random() - 0.5) * 2
        }
      })
    }
  })
}

// 更新EQ可视化
function updateEQVisualization() {
  if (!globalPlaying.value) return
  
  tracks.value.forEach((track) => {
    if (!track.isMuted) {
      track.eqBands.forEach((band) => {
        // 模拟实时EQ变化
        const variation = (Math.random() - 0.5) * 10
        band.value = Math.max(5, Math.min(80, band.value + variation))
      })
    }
  })
}

// 鼠标事件处理
function handleMouseDown(event) {
  mouse.value.isDown = true
  mouse.value.lastX = event.clientX
  mouse.value.lastY = event.clientY
}

function handleMouseMove(event) {
  if (!mouse.value.isDown) return
  
  const deltaX = event.clientX - mouse.value.lastX
  const deltaY = event.clientY - mouse.value.lastY
  
  if (currentView.value === 'free') {
    camera.value.rotY += deltaX * mouse.value.sensitivity
    camera.value.rotX += deltaY * mouse.value.sensitivity
  }
  
  mouse.value.lastX = event.clientX
  mouse.value.lastY = event.clientY
}

function handleMouseUp() {
  mouse.value.isDown = false
}

function handleWheel(event) {
  event.preventDefault()
  camera.value.z += event.deltaY * 0.002
  camera.value.z = Math.max(2, Math.min(15, camera.value.z))
}

// 窗口大小调整
function handleResize() {
  if (app && canvasContainer.value) {
    app.renderer.resize(canvasContainer.value.clientWidth, 500)
  }
}

// 控制函数
function toggleGlobalPlayback() {
  globalPlaying.value = !globalPlaying.value
  tracks.value.forEach(track => {
    track.isPlaying = globalPlaying.value && !track.isMuted
  })
}

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

function updateTrackVolume(trackId, volume) {
  const track = tracks.value.find(t => t.id === trackId)
  if (track) {
    track.volume = volume
  }
}

function updateMasterVolume(volume) {
  masterVolume.value = volume
  // 应用主音量到所有轨道
}

function setView(viewType) {
  currentView.value = viewType
  
  switch(viewType) {
    case 'front':
      camera.value.rotX = 0
      camera.value.rotY = 0
      camera.value.z = 8
      break
    case 'side':
      camera.value.rotX = 0
      camera.value.rotY = 90
      camera.value.z = 10
      break
    case 'top':
      camera.value.rotX = 90
      camera.value.rotY = 0
      camera.value.z = 12
      break
    case 'free':
      // 保持当前位置
      break
  }
}

// 生命周期
onMounted(async () => {
  await nextTick()
  await init3DScene()
})

onUnmounted(() => {
  if (app) {
    window.removeEventListener('resize', handleResize)
    app.destroy(true)
  }
})
</script>

<style scoped>
.enhanced-3d-waveform {
  width: 100%;
  background: linear-gradient(135deg, #0a0a0a 0%, #1a1a2e 50%, #16213e 100%);
  border-radius: 16px;
  overflow: hidden;
  border: 2px solid #333;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
}

.controls-panel {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px;
  background: rgba(255, 255, 255, 0.05);
  border-bottom: 2px solid #333;
  backdrop-filter: blur(10px);
}

.panel-left h3 {
  color: #fff;
  margin: 0 0 12px 0;
  font-size: 18px;
  font-weight: bold;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.5);
}

.view-controls {
  display: flex;
  gap: 8px;
}

.master-controls {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
}

.master-volume {
  display: flex;
  align-items: center;
  gap: 12px;
  color: #fff;
  font-size: 14px;
}

.effect-controls {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.canvas-3d-container {
  width: 100%;
  height: 500px;
  position: relative;
  cursor: grab;
  background: radial-gradient(ellipse at center, #1a1a2e 0%, #0a0a0a 100%);
}

.canvas-3d-container:active {
  cursor: grabbing;
}

.loading-indicator {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: #fff;
}

.tracks-control-panel {
  padding: 20px;
  background: rgba(0, 0, 0, 0.4);
  border-top: 2px solid #333;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 16px;
}

.track-controller {
  background: rgba(255, 255, 255, 0.08);
  border-radius: 12px;
  padding: 16px;
  border: 2px solid transparent;
  transition: all 0.3s ease;
}

.track-controller:hover {
  background: rgba(255, 255, 255, 0.12);
  transform: translateY(-2px);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.2);
}

.track-controller.active {
  border-color: #409eff;
  background: rgba(64, 158, 255, 0.15);
}

.track-controller.muted {
  opacity: 0.5;
  filter: grayscale(0.7);
}

.track-controller.solo {
  border-color: #e6a23c;
  background: rgba(230, 162, 60, 0.15);
}

.track-header {
  display: flex;
  align-items: center;
  margin-bottom: 16px;
}

.track-icon {
  width: 48px;
  height: 48px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  font-weight: bold;
  font-size: 16px;
  margin-right: 16px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
}

.track-info h4 {
  color: #fff;
  margin: 0 0 4px 0;
  font-size: 16px;
  font-weight: 600;
}

.track-stats {
  color: #999;
  font-size: 12px;
}

.track-controls {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.volume-control {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1;
  margin-right: 16px;
}

.volume-label {
  color: #fff;
  font-size: 12px;
  min-width: 32px;
}

.track-buttons {
  display: flex;
  gap: 8px;
}

.track-effects {
  margin-top: 16px;
}

.eq-visualizer {
  display: flex;
  align-items: end;
  height: 60px;
  gap: 2px;
  background: rgba(0, 0, 0, 0.3);
  padding: 8px;
  border-radius: 8px;
}

.eq-band {
  flex: 1;
  min-height: 4px;
  border-radius: 2px;
  transition: all 0.1s ease;
}
</style>