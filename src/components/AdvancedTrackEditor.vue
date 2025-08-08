<template>
  <div class="advanced-track-editor">
    <!-- 控制面板 -->
    <div class="controls-panel">
      <div class="panel-left">
        <h3>🎵 PixiJS轨道编辑器</h3>
        <div class="playback-controls">
          <t-button 
            size="large"
            :theme="isPlaying ? 'danger' : 'primary'"
            @click="togglePlayback"
          >
            {{ isPlaying ? '⏸️ 暂停' : '▶️ 播放' }}
          </t-button>
          <span class="time-display">{{ formatTime(currentTime) }}</span>
        </div>
      </div>
      
      <div class="panel-center">
        <div class="zoom-controls">
          <span>水平缩放:</span>
          <t-slider 
            v-model="zoomX" 
            :min="0.1" 
            :max="5" 
            :step="0.1"
            @change="updateZoom"
          />
          <span>垂直缩放:</span>
          <t-slider 
            v-model="zoomY" 
            :min="0.5" 
            :max="3" 
            :step="0.1"
            @change="updateZoom"
          />
        </div>
      </div>
      
      <div class="panel-right">
        <div class="view-info">
          <span>时间: {{ formatTime(viewportStartTime) }} - {{ formatTime(viewportEndTime) }}</span>
          <span>轨道: {{ tracks.length }}</span>
          <span>选中: {{ selectedClip?.name || '无' }}</span>
        </div>
      </div>
    </div>

    <!-- 主工作区域 -->
    <div class="main-workspace">
      <!-- 轨道信息侧边栏 -->
      <div class="track-sidebar">
        <!-- 时间轴占位区域 -->
        <div class="timeline-placeholder">
          <span>时间轴</span>
        </div>
        
        <!-- 轨道信息列表 -->
        <div class="tracks-sidebar-content" :style="{ transform: `translateY(-${scrollY}px)` }">
          <div 
            v-for="(track, index) in tracks" 
            :key="track.id"
            class="track-info"
            :style="{ 
              top: (index * trackHeight * zoomY.value) + 'px',
              height: (trackHeight * zoomY.value) + 'px'
            }"
          >
            <div class="track-content">
              <div class="track-header">
                <h4 :style="{ color: track.color }">{{ track.name }}</h4>
                <span class="track-type">{{ track.type }}</span>
              </div>
              
              <div class="track-controls">
                <t-button 
                  size="small" 
                  :theme="track.isSolo ? 'warning' : 'default'"
                  @click="toggleSolo(track.id)"
                >
                  S
                </t-button>
                <t-button 
                  size="small" 
                  :theme="track.isMuted ? 'danger' : 'default'"
                  @click="toggleMute(track.id)"
                >
                  M
                </t-button>
              </div>
              
              <div class="volume-control">
                <t-slider 
                  v-model="track.volume" 
                  :min="0" 
                  :max="100"
                  size="small"
                  vertical
                />
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- PixiJS渲染区域 -->
      <div class="pixi-workspace">
        <!-- 时间轴Canvas -->
        <div class="timeline-container">
          <canvas ref="timelineCanvas" class="timeline-canvas"></canvas>
        </div>
        
        <!-- 主PixiJS容器 -->
        <div 
          ref="pixiContainer" 
          class="pixi-container"
          @wheel="handleWheel"
          @mousedown="handleMouseDown"
          @mousemove="handleMouseMove"
          @mouseup="handleMouseUp"
          @mouseleave="handleMouseUp"
        >
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick, computed, reactive } from 'vue'
import * as PIXI from 'pixi.js'

// 响应式状态
const isPlaying = ref(false)
const currentTime = ref(0)
const zoomX = ref(1)
const zoomY = ref(1)
const scrollX = ref(0)
const scrollY = ref(0)
const selectedClip = ref(null)

// DOM引用
const pixiContainer = ref(null)
const timelineCanvas = ref(null)

// 布局常量
const trackHeight = 120
const timelineHeight = 60
const pixelsPerSecond = 30
const maxDuration = 80
const sidebarWidth = 200

// PixiJS相关变量
let app = null
let mainContainer = null
let tracksContainer = null
let playheadLine = null
let timelineCtx = null

// 性能优化：拖拽缓存
let clipGraphicsCache = new Map() // 缓存clip图形对象
let dragPreviewGraphics = null    // 拖拽预览图形
let lastRenderTime = 0           // 最后渲染时间
const RENDER_THROTTLE = 16       // 渲染节流 (60fps)

// 鼠标和拖拽状态
const mouse = reactive({
  isDown: false,
  lastX: 0,
  lastY: 0,
  isDragging: false
})

const clipDrag = reactive({
  isDragging: false,
  draggedClip: null,
  draggedClipGraphics: null,  // 被拖拽的clip图形对象
  startX: 0,
  startTime: 0,
  offsetX: 0,
  lastUpdateTime: 0,          // 最后更新时间
  pendingUpdate: false        // 是否有待处理的更新
})

// 计算属性
const viewportStartTime = computed(() => scrollX.value / (pixelsPerSecond * zoomX.value))
const viewportEndTime = computed(() => {
  const containerWidth = (pixiContainer.value?.clientWidth || 800)
  return viewportStartTime.value + (containerWidth / (pixelsPerSecond * zoomX.value))
})

// 轨道数据
const tracks = ref([
  {
    id: 1,
    name: '主旋律',
    type: 'LEAD',
    color: '#10b981',
    volume: 85,
    isSolo: false,
    isMuted: false,
    clips: [
      { id: 1, name: '主旋律_1', startTime: 0, duration: 15, color: '#10b981', waveformData: [] },
      { id: 2, name: '主旋律_2', startTime: 20, duration: 12, color: '#059669', waveformData: [] },
      { id: 3, name: '主旋律_3', startTime: 40, duration: 18, color: '#047857', waveformData: [] }
    ]
  },
  {
    id: 2,
    name: '鼓点',
    type: 'PERC',
    color: '#f59e0b',
    volume: 95,
    isSolo: false,
    isMuted: false,
    clips: [
      { id: 4, name: '鼓点_1', startTime: 0, duration: 8, color: '#f59e0b', waveformData: [] },
      { id: 5, name: '鼓点_2', startTime: 12, duration: 10, color: '#d97706', waveformData: [] },
      { id: 6, name: '鼓点_3', startTime: 25, duration: 8, color: '#f59e0b', waveformData: [] },
      { id: 7, name: '鼓点_4', startTime: 40, duration: 12, color: '#b45309', waveformData: [] }
    ]
  },
  {
    id: 3,
    name: '贝斯',
    type: 'BASS',
    color: '#3b82f6',
    volume: 70,
    isSolo: false,
    isMuted: false,
    clips: [
      { id: 8, name: '贝斯_1', startTime: 0, duration: 25, color: '#3b82f6', waveformData: [] },
      { id: 9, name: '贝斯_2', startTime: 30, duration: 20, color: '#2563eb', waveformData: [] }
    ]
  },
  {
    id: 4,
    name: '和弦',
    type: 'CHORD',
    color: '#8b5cf6',
    volume: 60,
    isSolo: false,
    isMuted: false,
    clips: [
      { id: 10, name: '和弦_1', startTime: 5, duration: 18, color: '#8b5cf6', waveformData: [] },
      { id: 11, name: '和弦_2', startTime: 30, duration: 15, color: '#7c3aed', waveformData: [] },
      { id: 12, name: '和弦_3', startTime: 50, duration: 20, color: '#6d28d9', waveformData: [] }
    ]
  },
  {
    id: 5,
    name: '合成器',
    type: 'SYNTH',
    color: '#ef4444',
    volume: 75,
    isSolo: false,
    isMuted: false,
    clips: [
      { id: 13, name: '合成器_1', startTime: 10, duration: 20, color: '#ef4444', waveformData: [] },
      { id: 14, name: '合成器_2', startTime: 35, duration: 15, color: '#dc2626', waveformData: [] }
    ]
  },
  {
    id: 6,
    name: '人声',
    type: 'VOCAL',
    color: '#f97316',
    volume: 90,
    isSolo: false,
    isMuted: false,
    clips: [
      { id: 15, name: '主歌', startTime: 8, duration: 16, color: '#f97316', waveformData: [] },
      { id: 16, name: '副歌', startTime: 30, duration: 20, color: '#ea580c', waveformData: [] },
      { id: 17, name: '桥段', startTime: 55, duration: 10, color: '#c2410c', waveformData: [] }
    ]
  }
])

// 生成波形数据
function generateWaveformData(clip, trackType) {
  const pointsPerSecond = 50
  const totalPoints = clip.duration * pointsPerSecond
  const waveform = []
  
  for (let i = 0; i < totalPoints; i++) {
    const time = i / pointsPerSecond
    let amplitude = 0
    
    switch(trackType) {
      case 'LEAD':
        amplitude = Math.sin(time * 4 + clip.startTime) * 0.8 + Math.sin(time * 8 + clip.startTime) * 0.3
        break
      case 'BASS':
        amplitude = Math.sign(Math.sin(time * 2 + clip.startTime)) * 0.9
        break
      case 'PERC':
        const beat = Math.floor((time + clip.startTime) * 4) % 4
        if (beat === 0) {
          amplitude = Math.exp(-((time % 1) * 8)) * 1.2
        } else if (beat === 2) {
          amplitude = Math.exp(-((time % 1) * 6)) * 0.8
        }
        break
      case 'CHORD':
        amplitude = Math.sin(time * 3 + Math.sin((time + clip.startTime) * 0.5)) * 0.7
        break
      case 'SYNTH':
        amplitude = Math.sin(time * 6 + clip.startTime) * 0.6 + Math.sin(time * 2.5 + clip.startTime) * 0.4
        break
      case 'VOCAL':
        amplitude = Math.sin(time * 2 + clip.startTime) * 0.4 + Math.sin(time * 2.5 + clip.startTime) * 0.3
        break
      default:
        amplitude = Math.sin(time * 4 + clip.startTime) * 0.6
    }
    
    amplitude = Math.max(0, Math.min(1, Math.abs(amplitude)))
    waveform.push(amplitude)
  }
  
  return waveform
}

// 初始化轨道数据
function initializeTracks() {
  tracks.value.forEach(track => {
    track.clips.forEach(clip => {
      if (!clip.waveformData.length) {
        clip.waveformData = generateWaveformData(clip, track.type)
      }
    })
  })
}

// 初始化PixiJS
async function initPixi() {
  const container = pixiContainer.value
  if (!container) return

  const width = container.clientWidth || 800
  const height = container.clientHeight || 600

  try {
    app = new PIXI.Application({
      width,
      height,
      backgroundColor: 0x1a1a1a,
      antialias: true,
      resolution: window.devicePixelRatio || 1,
      autoDensity: true
    })
    
    container.appendChild(app.view)
    
    // 创建主容器
    mainContainer = new PIXI.Container()
    app.stage.addChild(mainContainer)
    
    // 创建轨道容器
    tracksContainer = new PIXI.Container()
    tracksContainer.y = 0
    mainContainer.addChild(tracksContainer)
    
    // 创建播放头
    playheadLine = new PIXI.Graphics()
    playheadLine.lineStyle(2, 0xff4444)
    playheadLine.moveTo(0, 0)
    playheadLine.lineTo(0, height)
    mainContainer.addChild(playheadLine)
    
    // 初始化轨道数据
    initializeTracks()
    
    // 创建轨道
    createTracks()
    
    // 初始化时间轴Canvas
    initTimelineCanvas()
    
    console.log('✅ PixiJS轨道编辑器初始化成功')
  } catch (error) {
    console.error('❌ PixiJS初始化失败:', error)
  }
}

// 初始化时间轴Canvas
function initTimelineCanvas() {
  const canvas = timelineCanvas.value
  if (!canvas) return
  
  const containerWidth = pixiContainer.value?.clientWidth || 800
  canvas.width = containerWidth * (window.devicePixelRatio || 1)
  canvas.height = timelineHeight * (window.devicePixelRatio || 1)
  canvas.style.width = containerWidth + 'px'
  canvas.style.height = timelineHeight + 'px'
  
  timelineCtx = canvas.getContext('2d')
  timelineCtx.scale(window.devicePixelRatio || 1, window.devicePixelRatio || 1)
  
  drawTimeline()
}

// 绘制时间轴
function drawTimeline() {
  if (!timelineCtx) return
  
  const canvas = timelineCanvas.value
  const width = canvas.width / (window.devicePixelRatio || 1)
  const height = canvas.height / (window.devicePixelRatio || 1)
  
  // 清除画布
  timelineCtx.clearRect(0, 0, width, height)
  
  // 背景
  timelineCtx.fillStyle = '#2a2a2a'
  timelineCtx.fillRect(0, 0, width, height)
  
  // 计算可视时间范围
  const startTime = Math.floor(viewportStartTime.value)
  const endTime = Math.ceil(viewportEndTime.value)
  
  // 绘制时间刻度
  timelineCtx.strokeStyle = '#666666'
  timelineCtx.fillStyle = '#ffffff'
  timelineCtx.font = '12px Arial'
  timelineCtx.textAlign = 'center'
  
  for (let t = startTime; t <= endTime; t++) {
    const x = (t * pixelsPerSecond * zoomX.value) - scrollX.value
    
    if (x >= -50 && x <= width + 50) {
      // 主刻度线
      timelineCtx.beginPath()
      timelineCtx.moveTo(x, height - 20)
      timelineCtx.lineTo(x, height)
      timelineCtx.stroke()
      
      // 时间标签
      if (t % Math.max(1, Math.floor(2 / zoomX.value)) === 0) {
        timelineCtx.fillText(`${t}s`, x, height - 25)
      }
      
      // 次要刻度线
      if (zoomX.value > 0.5) {
        for (let subT = 0.2; subT < 1; subT += 0.2) {
          const subX = ((t + subT) * pixelsPerSecond * zoomX.value) - scrollX.value
          if (subX >= 0 && subX <= width) {
            timelineCtx.strokeStyle = '#444444'
            timelineCtx.beginPath()
            timelineCtx.moveTo(subX, height - 10)
            timelineCtx.lineTo(subX, height)
            timelineCtx.stroke()
            timelineCtx.strokeStyle = '#666666'
          }
        }
      }
    }
  }
  
  // 绘制播放头
  const playheadX = (currentTime.value * pixelsPerSecond * zoomX.value) - scrollX.value
  if (playheadX >= 0 && playheadX <= width) {
    timelineCtx.strokeStyle = '#ff4444'
    timelineCtx.lineWidth = 2
    timelineCtx.beginPath()
    timelineCtx.moveTo(playheadX, 0)
    timelineCtx.lineTo(playheadX, height)
    timelineCtx.stroke()
    timelineCtx.lineWidth = 1
  }
}

// 节流函数
function throttle(func, delay) {
  let timeoutId
  let lastExecTime = 0
  return function (...args) {
    const currentTime = Date.now()
    
    if (currentTime - lastExecTime > delay) {
      func.apply(this, args)
      lastExecTime = currentTime
    } else {
      clearTimeout(timeoutId)
      timeoutId = setTimeout(() => {
        func.apply(this, args)
        lastExecTime = Date.now()
      }, delay - (currentTime - lastExecTime))
    }
  }
}

// 创建轨道（优化版本）
function createTracks() {
  if (!tracksContainer) return
  
  tracksContainer.removeChildren()
  clipGraphicsCache.clear() // 清除缓存
  
  tracks.value.forEach((track, index) => {
    const trackContainer = new PIXI.Container()
    trackContainer.y = index * trackHeight * zoomY.value
    
    // 轨道背景
    const trackBg = new PIXI.Graphics()
    trackBg.beginFill(index % 2 ? 0x1e1e1e : 0x252525)
    trackBg.drawRect(0, 0, maxDuration * pixelsPerSecond * zoomX.value, trackHeight * zoomY.value)
    trackBg.endFill()
    trackContainer.addChild(trackBg)
    
    // 轨道分割线
    const separator = new PIXI.Graphics()
    separator.lineStyle(1, 0x333333)
    separator.moveTo(0, trackHeight * zoomY.value)
    separator.lineTo(maxDuration * pixelsPerSecond * zoomX.value, trackHeight * zoomY.value)
    trackContainer.addChild(separator)
    
    // 创建clips
    track.clips.forEach(clip => {
      const clipContainer = createClip(clip, track, index)
      clipGraphicsCache.set(clip.id, clipContainer) // 缓存clip图形
      trackContainer.addChild(clipContainer)
    })
    
    tracksContainer.addChild(trackContainer)
  })
  
  // 重置轨道容器位置
  tracksContainer.y = 0
}

// 高性能轨道重绘（仅更新位置）
function updateClipPositions() {
  tracks.value.forEach((track, trackIndex) => {
    track.clips.forEach(clip => {
      const clipContainer = clipGraphicsCache.get(clip.id)
      if (clipContainer && clipContainer.children.length > 0) {
        const newClipX = clip.startTime * pixelsPerSecond * zoomX.value
        
        // 直接使用完整重绘来确保波形跟随
        recreateClip(clip, track, trackIndex, clipContainer)
      }
    })
  })
}

// 重新创建单个clip（保持容器，重绘内容）
function recreateClip(clip, track, trackIndex, existingContainer) {
  // 清空现有内容
  existingContainer.removeChildren()
  
  const clipWidth = clip.duration * pixelsPerSecond * zoomX.value
  const clipHeight = trackHeight * zoomY.value - 20
  const clipX = clip.startTime * pixelsPerSecond * zoomX.value
  
  // Clip背景
  const clipBg = new PIXI.Graphics()
  const color = parseInt(clip.color.replace('#', ''), 16)
  clipBg.beginFill(color, 0.8)
  clipBg.lineStyle(2, color, 1)
  clipBg.drawRoundedRect(clipX, 10, clipWidth, clipHeight, 6)
  clipBg.endFill()
  
  // 设置交互
  clipBg.interactive = true
  clipBg.buttonMode = true
  clipBg.clip = clip
  clipBg.track = track
  
  clipBg.on('pointerdown', (event) => handleClipMouseDown(event, clip, track, existingContainer))
  clipBg.on('pointerover', () => clipBg.alpha = 0.9)
  clipBg.on('pointerout', () => clipBg.alpha = 1.0)
  
  existingContainer.addChild(clipBg)
  
  // 创建波形（跟随clip位置）
  if (clip.waveformData && clip.waveformData.length > 0) {
    const waveform = createWaveform(clip, clipX, clipWidth, clipHeight)
    existingContainer.addChild(waveform)
  }
  
  // Clip标题
  const clipText = new PIXI.Text(clip.name, {
    fontSize: Math.max(10, 12 * zoomY.value),
    fill: 0xffffff,
    fontWeight: 'bold'
  })
  clipText.x = clipX + 8
  clipText.y = 15
  existingContainer.addChild(clipText)
  
  // Clip时间信息
  const timeText = new PIXI.Text(`${clip.duration.toFixed(1)}s`, {
    fontSize: Math.max(8, 10 * zoomY.value),
    fill: 0xcccccc
  })
  timeText.x = clipX + 8
  timeText.y = clipHeight - 5
  existingContainer.addChild(timeText)
}

// 重绘波形（优化版本）
function redrawWaveform(waveformGraphics, clip, clipX, clipWidth, clipHeight) {
  const waveformData = clip.waveformData
  if (!waveformData || waveformData.length === 0) return
  
  waveformGraphics.clear()
  
  const pointWidth = clipWidth / waveformData.length
  const color = parseInt(clip.color.replace('#', ''), 16)
  
  waveformGraphics.beginFill(color, 0.4)
  
  const baselineY = clipHeight / 2 + 10
  const amplitudeScale = (clipHeight - 30) / 4
  
  // 绘制波形上半部分
  waveformGraphics.moveTo(clipX, baselineY)
  for (let i = 0; i < waveformData.length; i++) {
    const x = clipX + i * pointWidth
    const amplitude = waveformData[i] * amplitudeScale
    waveformGraphics.lineTo(x, baselineY - amplitude)
  }
  
  // 绘制波形下半部分
  for (let i = waveformData.length - 1; i >= 0; i--) {
    const x = clipX + i * pointWidth
    const amplitude = waveformData[i] * amplitudeScale
    waveformGraphics.lineTo(x, baselineY + amplitude)
  }
  
  waveformGraphics.closePath()
  waveformGraphics.endFill()
}

// 创建clip（优化版本）
function createClip(clip, track, trackIndex) {
  const clipContainer = new PIXI.Container()
  clipContainer.clipId = clip.id // 标记ID便于查找
  
  const clipWidth = clip.duration * pixelsPerSecond * zoomX.value
  const clipHeight = trackHeight * zoomY.value - 20
  const clipX = clip.startTime * pixelsPerSecond * zoomX.value
  
  // Clip背景
  const clipBg = new PIXI.Graphics()
  const color = parseInt(clip.color.replace('#', ''), 16)
  clipBg.beginFill(color, 0.8)
  clipBg.lineStyle(2, color, 1)
  clipBg.drawRoundedRect(clipX, 10, clipWidth, clipHeight, 6)
  clipBg.endFill()
  clipBg.originalX = clipX // 记录原始X位置
  
  // 设置交互
  clipBg.interactive = true
  clipBg.buttonMode = true
  clipBg.clip = clip
  clipBg.track = track
  
  clipBg.on('pointerdown', (event) => handleClipMouseDown(event, clip, track, clipContainer))
  clipBg.on('pointerover', () => clipBg.alpha = 0.9)
  clipBg.on('pointerout', () => clipBg.alpha = 1.0)
  
  clipContainer.addChild(clipBg)
  
  // 创建波形
  if (clip.waveformData && clip.waveformData.length > 0) {
    const waveform = createWaveform(clip, clipX, clipWidth, clipHeight)
    waveform.originalX = clipX // 记录原始X位置
    clipContainer.addChild(waveform)
  }
  
  // Clip标题
  const clipText = new PIXI.Text(clip.name, {
    fontSize: Math.max(10, 12 * zoomY.value),
    fill: 0xffffff,
    fontWeight: 'bold'
  })
  clipText.x = clipX + 8
  clipText.y = 15
  clipText.originalX = clipX + 8 // 记录原始X位置
  clipContainer.addChild(clipText)
  
  // Clip时间信息
  const timeText = new PIXI.Text(`${clip.duration.toFixed(1)}s`, {
    fontSize: Math.max(8, 10 * zoomY.value),
    fill: 0xcccccc
  })
  timeText.x = clipX + 8
  timeText.y = clipHeight - 5
  timeText.originalX = clipX + 8 // 记录原始X位置
  clipContainer.addChild(timeText)
  
  return clipContainer
}

// 创建波形
function createWaveform(clip, clipX, clipWidth, clipHeight) {
  const waveformContainer = new PIXI.Container()
  const waveformData = clip.waveformData
  
  if (!waveformData || waveformData.length === 0) return waveformContainer
  
  const pointWidth = clipWidth / waveformData.length
  const color = parseInt(clip.color.replace('#', ''), 16)
  
  const waveform = new PIXI.Graphics()
  waveform.beginFill(color, 0.4)
  
  const baselineY = clipHeight / 2 + 10
  const amplitudeScale = (clipHeight - 30) / 4
  
  // 绘制波形上半部分
  waveform.moveTo(clipX, baselineY)
  for (let i = 0; i < waveformData.length; i++) {
    const x = clipX + i * pointWidth
    const amplitude = waveformData[i] * amplitudeScale
    waveform.lineTo(x, baselineY - amplitude)
  }
  
  // 绘制波形下半部分
  for (let i = waveformData.length - 1; i >= 0; i--) {
    const x = clipX + i * pointWidth
    const amplitude = waveformData[i] * amplitudeScale
    waveform.lineTo(x, baselineY + amplitude)
  }
  
  waveform.closePath()
  waveform.endFill()
  
  waveformContainer.addChild(waveform)
  return waveformContainer
}

// 节流的轨道更新函数
const throttledUpdateClipPositions = throttle(updateClipPositions, RENDER_THROTTLE)

// Clip拖拽事件（优化版本）
function handleClipMouseDown(event, clip, track, clipContainer) {
  event.stopPropagation()
  
  clipDrag.isDragging = true
  clipDrag.draggedClip = clip
  clipDrag.draggedClipGraphics = clipContainer
  clipDrag.startX = event.data.global.x
  clipDrag.startTime = clip.startTime
  clipDrag.offsetX = 0
  clipDrag.lastUpdateTime = Date.now()
  clipDrag.pendingUpdate = false
  
  selectedClip.value = clip
  
  // 创建拖拽预览（半透明副本）
  if (!dragPreviewGraphics) {
    dragPreviewGraphics = new PIXI.Graphics()
    dragPreviewGraphics.alpha = 0.5
    tracksContainer.addChild(dragPreviewGraphics)
  }
  
  document.addEventListener('pointermove', handleClipDragOptimized)
  document.addEventListener('pointerup', stopClipDrag)
  
  console.log('开始拖拽片段:', clip.name)
}

// 优化的拖拽处理函数
function handleClipDragOptimized(event) {
  if (!clipDrag.isDragging) return
  
  const currentTime = Date.now()
  
  // 节流：限制更新频率
  if (currentTime - clipDrag.lastUpdateTime < RENDER_THROTTLE) {
    if (!clipDrag.pendingUpdate) {
      clipDrag.pendingUpdate = true
      requestAnimationFrame(() => {
        if (clipDrag.isDragging) {
          updateDragPosition(event)
        }
        clipDrag.pendingUpdate = false
      })
    }
    return
  }
  
  updateDragPosition(event)
  clipDrag.lastUpdateTime = currentTime
}

// 更新拖拽位置（分离的函数）
function updateDragPosition(event) {
  const deltaX = event.clientX - clipDrag.startX
  const newTime = clipDrag.startTime + (deltaX / (pixelsPerSecond * zoomX.value))
  const snapTime = Math.max(0, Math.round(newTime * 4) / 4) // 1/4秒对齐
  
  if (clipDrag.draggedClip && Math.abs(clipDrag.draggedClip.startTime - snapTime) > 0.01) {
    clipDrag.draggedClip.startTime = snapTime
    
    // 使用快速位置更新而不是完全重绘
    throttledUpdateClipPositions()
    
    // 更新拖拽预览位置
    if (dragPreviewGraphics && clipDrag.draggedClipGraphics) {
      const newX = snapTime * pixelsPerSecond * zoomX.value
      dragPreviewGraphics.clear()
      
      // 绘制预览框
      const clipWidth = clipDrag.draggedClip.duration * pixelsPerSecond * zoomX.value
      const clipHeight = trackHeight * zoomY.value - 20
      const color = parseInt(clipDrag.draggedClip.color.replace('#', ''), 16)
      
      dragPreviewGraphics.lineStyle(2, color, 0.8)
      dragPreviewGraphics.beginFill(color, 0.2)
      dragPreviewGraphics.drawRoundedRect(newX, clipDrag.draggedClipGraphics.y + 10, clipWidth, clipHeight, 6)
      dragPreviewGraphics.endFill()
    }
  }
}

function stopClipDrag() {
  if (clipDrag.isDragging) {
    console.log('停止拖拽片段:', clipDrag.draggedClip?.name, '新位置:', clipDrag.draggedClip?.startTime)
    
    // 清除拖拽预览
    if (dragPreviewGraphics) {
      dragPreviewGraphics.clear()
    }
    
    // 最终重绘确保一致性
    createTracks()
  }
  
  clipDrag.isDragging = false
  clipDrag.draggedClip = null
  clipDrag.draggedClipGraphics = null
  clipDrag.pendingUpdate = false
  
  document.removeEventListener('pointermove', handleClipDragOptimized)
  document.removeEventListener('pointerup', stopClipDrag)
}

// 滚轮事件
function handleWheel(event) {
  event.preventDefault()
  
  if (event.ctrlKey || event.metaKey) {
    // 缩放操作
    const zoomFactor = event.deltaY > 0 ? 0.9 : 1.1
    if (event.shiftKey) {
      zoomY.value = Math.max(0.5, Math.min(3, zoomY.value * zoomFactor))
    } else {
      zoomX.value = Math.max(0.1, Math.min(5, zoomX.value * zoomFactor))
    }
    updateZoom()
  } else {
    // 滚动操作
    const scrollSpeed = 50 // 增加滚动速度
    
    if (event.shiftKey) {
      // 水平滚动
      const containerWidth = pixiContainer.value?.clientWidth || 800
      const contentWidth = maxDuration * pixelsPerSecond * zoomX.value
      const maxScrollX = Math.max(0, contentWidth - containerWidth)
      
      const deltaX = event.deltaY > 0 ? scrollSpeed : -scrollSpeed
      scrollX.value = Math.max(0, Math.min(maxScrollX, scrollX.value + deltaX))
    } else {
      // 垂直滚动
      const containerHeight = pixiContainer.value?.clientHeight || 400
      const contentHeight = tracks.value.length * trackHeight * zoomY.value
      const maxScrollY = Math.max(0, contentHeight - containerHeight)
      
      const deltaY = event.deltaY > 0 ? scrollSpeed : -scrollSpeed
      scrollY.value = Math.max(0, Math.min(maxScrollY, scrollY.value + deltaY))
    }
    updateViewport()
  }
}

// 鼠标事件
function handleMouseDown(event) {
  if (clipDrag.isDragging) return
  
  mouse.isDown = true
  mouse.lastX = event.clientX
  mouse.lastY = event.clientY
  mouse.isDragging = false
}

function handleMouseMove(event) {
  if (clipDrag.isDragging) return
  
  if (mouse.isDown) {
    const deltaX = event.clientX - mouse.lastX
    const deltaY = event.clientY - mouse.lastY
    
    if (!mouse.isDragging && (Math.abs(deltaX) > 3 || Math.abs(deltaY) > 3)) {
      mouse.isDragging = true
    }
    
    if (mouse.isDragging) {
      const containerWidth = pixiContainer.value?.clientWidth || 800
      const containerHeight = pixiContainer.value?.clientHeight || 400
      const contentWidth = maxDuration * pixelsPerSecond * zoomX.value
      const contentHeight = tracks.value.length * trackHeight * zoomY.value
      
      const maxScrollX = Math.max(0, contentWidth - containerWidth)
      const maxScrollY = Math.max(0, contentHeight - containerHeight)
      
      scrollX.value = Math.max(0, Math.min(maxScrollX, scrollX.value - deltaX))
      scrollY.value = Math.max(0, Math.min(maxScrollY, scrollY.value - deltaY))
      
      updateViewport()
    }
    
    mouse.lastX = event.clientX
    mouse.lastY = event.clientY
  }
}

function handleMouseUp() {
  if (!clipDrag.isDragging) {
    mouse.isDown = false
    mouse.isDragging = false
  }
}

// 更新缩放（优化版本）
function updateZoom() {
  createTracks()
  drawTimeline()
  updatePlayhead()
}

// 更新视口
function updateViewport() {
  if (mainContainer) {
    mainContainer.x = -scrollX.value
    mainContainer.y = -scrollY.value
  }
  
  // 确保轨道容器正确定位
  if (tracksContainer) {
    tracksContainer.y = -scrollY.value
  }
  
  updatePlayhead()
  drawTimeline()
}

// 更新播放头
function updatePlayhead() {
  if (playheadLine) {
    playheadLine.x = currentTime.value * pixelsPerSecond * zoomX.value
  }
}

// 播放控制
let playInterval = null

function togglePlayback() {
  isPlaying.value = !isPlaying.value
  
  if (isPlaying.value) {
    playInterval = setInterval(() => {
      currentTime.value += 0.1
      if (currentTime.value >= maxDuration) {
        currentTime.value = 0
      }
      updatePlayhead()
      drawTimeline()
    }, 100)
  } else {
    if (playInterval) {
      clearInterval(playInterval)
      playInterval = null
    }
  }
}

// 轨道控制
function toggleSolo(trackId) {
  const track = tracks.value.find(t => t.id === trackId)
  if (track) {
    track.isSolo = !track.isSolo
    if (track.isSolo) {
      tracks.value.forEach(t => {
        if (t.id !== trackId) t.isMuted = true
      })
    } else {
      tracks.value.forEach(t => t.isMuted = false)
    }
    createTracks()
  }
}

function toggleMute(trackId) {
  const track = tracks.value.find(t => t.id === trackId)
  if (track) {
    track.isMuted = !track.isMuted
    createTracks()
  }
}

// 窗口大小调整
function handleResize() {
  if (app && app.renderer && pixiContainer.value) {
    const newWidth = pixiContainer.value.clientWidth || 800
    const newHeight = pixiContainer.value.clientHeight || 600
    
    app.renderer.resize(newWidth, newHeight)
    createTracks()
    initTimelineCanvas()
    updatePlayhead()
  }
}

// 工具函数
function formatTime(seconds) {
  const mins = Math.floor(seconds / 60)
  const secs = Math.floor(seconds % 60)
  return `${mins}:${secs.toString().padStart(2, '0')}`
}

// 生命周期
onMounted(async () => {
  await nextTick()
  await initPixi()
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  if (playInterval) {
    clearInterval(playInterval)
  }
  if (app) {
    window.removeEventListener('resize', handleResize)
    app.destroy(true)
  }
  
  // 清理拖拽相关的事件监听器和缓存
  document.removeEventListener('pointermove', handleClipDragOptimized)
  document.removeEventListener('pointerup', stopClipDrag)
  clipGraphicsCache.clear()
  dragPreviewGraphics = null
})
</script>

<style scoped>
.advanced-track-editor {
  width: 100%;
  height: 600px;
  background: #0a0a0a;
  border-radius: 16px;
  overflow: hidden;
  border: 2px solid #333;
  position: relative;
}

.controls-panel {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  background: rgba(255, 255, 255, 0.05);
  border-bottom: 2px solid #333;
  height: 80px;
  box-sizing: border-box;
}

.panel-left h3 {
  margin: 0 0 10px 0;
  color: #fff;
  font-size: 16px;
}

.playback-controls {
  display: flex;
  align-items: center;
  gap: 15px;
}

.time-display {
  color: #10b981;
  font-family: 'Courier New', monospace;
  font-size: 16px;
  font-weight: bold;
}

.zoom-controls {
  display: flex;
  align-items: center;
  gap: 10px;
  color: #fff;
  font-size: 14px;
}

.zoom-controls .t-slider {
  width: 100px;
}

.view-info {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  color: #999;
  font-size: 12px;
  gap: 5px;
}

.main-workspace {
  display: flex;
  height: calc(100% - 80px);
  width: 100%;
}

.track-sidebar {
  position: relative;
  width: 200px;
  height: 100%;
  background: rgba(0, 0, 0, 0.8);
  backdrop-filter: blur(10px);
  border-right: 2px solid #333;
  overflow: hidden;
  pointer-events: auto;
  z-index: 10;
}

.timeline-placeholder {
  height: 60px;
  background: rgba(255, 255, 255, 0.02);
  border-bottom: 1px solid #333;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #666;
  font-size: 12px;
}

.tracks-sidebar-content {
  position: relative;
  overflow: hidden;
}

.track-info {
  position: absolute;
  width: 100%;
  padding: 15px;
  box-sizing: border-box;
  border-bottom: 1px solid #333;
  display: flex;
  align-items: center;
  gap: 10px;
  cursor: default;
  transition: background-color 0.2s;
}

.track-info:hover {
  background: rgba(255, 255, 255, 0.02);
}

.track-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.track-header h4 {
  margin: 0;
  font-size: 14px;
}

.track-type {
  font-size: 10px;
  background: rgba(255, 255, 255, 0.1);
  padding: 2px 6px;
  border-radius: 8px;
  color: #ccc;
  width: fit-content;
}

.track-controls {
  display: flex;
  gap: 8px;
}

.track-controls .t-button {
  width: 28px;
  height: 28px;
  font-weight: bold;
  font-size: 12px;
}

.volume-control {
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
}

.pixi-workspace {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.timeline-container {
  height: 60px;
  border-bottom: 1px solid #333;
  overflow: hidden;
}

.timeline-canvas {
  display: block;
  width: 100%;
  height: 100%;
}

.pixi-container {
  flex: 1;
  position: relative;
  overflow: hidden;
  cursor: grab;
}

.pixi-container:active {
  cursor: grabbing;
}
</style>