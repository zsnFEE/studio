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
          <div class="debug-info" style="position: absolute; top: -20px; left: 0; color: #fff; font-size: 10px; z-index: 100;">
            轨道数: {{ tracks.length }}, 缩放: {{ zoomY.value }}, 滚动: {{ scrollY }}
          </div>
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
                <div class="track-buttons">
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
                  <span class="volume-label">{{ track.volume }}</span>
                  <t-slider 
                    v-model="track.volume" 
                    :min="0" 
                    :max="100"
                    size="small"
                    :style="{ width: '40px' }"
                  />
                </div>
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
const RENDER_THROTTLE = 500      // 渲染节流 (2fps) - 多clip场景下进一步降低频率

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
  originalTrack: null,        // 原始轨道
  targetTrack: null,          // 目标轨道
  startX: 0,
  startY: 0,                  // 添加Y坐标
  startTime: 0,
  offsetX: 0,
  lastUpdateTime: 0,          // 最后更新时间
  lastPreviewUpdate: 0,       // 最后预览更新时间
  pendingUpdate: false,       // 是否有待处理的更新
  snapTime: 0,                // 停靠时间
  snapToClip: null           // 停靠到的clip
})

// 计算属性
const viewportStartTime = computed(() => scrollX.value / (pixelsPerSecond * zoomX.value))
const viewportEndTime = computed(() => {
  const containerWidth = (pixiContainer.value?.clientWidth || 800)
  return viewportStartTime.value + (containerWidth / (pixelsPerSecond * zoomX.value))
})

// 轨道数据 - 增加更多clip
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
      { id: 1, name: '主旋律_1', startTime: 0, duration: 8, color: '#10b981', waveformData: [] },
      { id: 2, name: '主旋律_2', startTime: 10, duration: 6, color: '#059669', waveformData: [] },
      { id: 3, name: '主旋律_3', startTime: 18, duration: 10, color: '#047857', waveformData: [] },
      { id: 4, name: '主旋律_4', startTime: 30, duration: 8, color: '#065f46', waveformData: [] },
      { id: 5, name: '主旋律_5', startTime: 40, duration: 12, color: '#064e3b', waveformData: [] },
      { id: 6, name: '主旋律_6', startTime: 55, duration: 9, color: '#134e4a', waveformData: [] },
      { id: 7, name: '主旋律_7', startTime: 67, duration: 11, color: '#0f766e', waveformData: [] }
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
      { id: 8, name: '鼓点_1', startTime: 0, duration: 4, color: '#f59e0b', waveformData: [] },
      { id: 9, name: '鼓点_2', startTime: 6, duration: 4, color: '#d97706', waveformData: [] },
      { id: 10, name: '鼓点_3', startTime: 12, duration: 6, color: '#b45309', waveformData: [] },
      { id: 11, name: '鼓点_4', startTime: 20, duration: 4, color: '#92400e', waveformData: [] },
      { id: 12, name: '鼓点_5', startTime: 26, duration: 8, color: '#78350f', waveformData: [] },
      { id: 13, name: '鼓点_6', startTime: 36, duration: 5, color: '#451a03', waveformData: [] },
      { id: 14, name: '鼓点_7', startTime: 43, duration: 7, color: '#f59e0b', waveformData: [] },
      { id: 15, name: '鼓点_8', startTime: 52, duration: 6, color: '#d97706', waveformData: [] },
      { id: 16, name: '鼓点_9', startTime: 60, duration: 8, color: '#b45309', waveformData: [] },
      { id: 17, name: '鼓点_10', startTime: 70, duration: 5, color: '#92400e', waveformData: [] }
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
      { id: 18, name: '贝斯_1', startTime: 0, duration: 12, color: '#3b82f6', waveformData: [] },
      { id: 19, name: '贝斯_2', startTime: 14, duration: 10, color: '#2563eb', waveformData: [] },
      { id: 20, name: '贝斯_3', startTime: 26, duration: 14, color: '#1d4ed8', waveformData: [] },
      { id: 21, name: '贝斯_4', startTime: 42, duration: 8, color: '#1e40af', waveformData: [] },
      { id: 22, name: '贝斯_5', startTime: 52, duration: 16, color: '#1e3a8a', waveformData: [] },
      { id: 23, name: '贝斯_6', startTime: 70, duration: 8, color: '#1d4ed8', waveformData: [] }
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
      { id: 24, name: '和弦_1', startTime: 2, duration: 10, color: '#8b5cf6', waveformData: [] },
      { id: 25, name: '和弦_2', startTime: 14, duration: 8, color: '#7c3aed', waveformData: [] },
      { id: 26, name: '和弦_3', startTime: 24, duration: 12, color: '#6d28d9', waveformData: [] },
      { id: 27, name: '和弦_4', startTime: 38, duration: 9, color: '#5b21b6', waveformData: [] },
      { id: 28, name: '和弦_5', startTime: 49, duration: 11, color: '#4c1d95', waveformData: [] },
      { id: 29, name: '和弦_6', startTime: 62, duration: 7, color: '#581c87', waveformData: [] },
      { id: 30, name: '和弦_7', startTime: 71, duration: 6, color: '#6b21a8', waveformData: [] }
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
      { id: 31, name: '合成器_1', startTime: 4, duration: 6, color: '#ef4444', waveformData: [] },
      { id: 32, name: '合成器_2', startTime: 12, duration: 8, color: '#dc2626', waveformData: [] },
      { id: 33, name: '合成器_3', startTime: 22, duration: 5, color: '#b91c1c', waveformData: [] },
      { id: 34, name: '合成器_4', startTime: 29, duration: 9, color: '#991b1b', waveformData: [] },
      { id: 35, name: '合成器_5', startTime: 40, duration: 7, color: '#7f1d1d', waveformData: [] },
      { id: 36, name: '合成器_6', startTime: 49, duration: 10, color: '#450a0a', waveformData: [] },
      { id: 37, name: '合成器_7', startTime: 61, duration: 6, color: '#ef4444', waveformData: [] },
      { id: 38, name: '合成器_8', startTime: 69, duration: 8, color: '#dc2626', waveformData: [] }
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
      { id: 39, name: '人声_1', startTime: 8, duration: 6, color: '#f97316', waveformData: [] },
      { id: 40, name: '人声_2', startTime: 16, duration: 8, color: '#ea580c', waveformData: [] },
      { id: 41, name: '人声_3', startTime: 26, duration: 5, color: '#c2410c', waveformData: [] },
      { id: 42, name: '人声_4', startTime: 33, duration: 7, color: '#9a3412', waveformData: [] },
      { id: 43, name: '人声_5', startTime: 42, duration: 9, color: '#7c2d12', waveformData: [] },
      { id: 44, name: '人声_6', startTime: 53, duration: 6, color: '#431407', waveformData: [] },
      { id: 45, name: '人声_7', startTime: 61, duration: 8, color: '#f97316', waveformData: [] },
      { id: 46, name: '人声_8', startTime: 71, duration: 5, color: '#ea580c', waveformData: [] }
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
    tracksContainer.x = 0
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

// 高性能拖拽更新策略
let dragUpdateTimeout = null
let isUpdating = false

function updateClipPositions() {
  // 防止重复更新
  if (isUpdating) return
  
  // 清除之前的延迟更新
  if (dragUpdateTimeout) {
    clearTimeout(dragUpdateTimeout)
  }
  
  // 进一步延长延迟，减少多clip场景下的卡顿
  dragUpdateTimeout = setTimeout(() => {
    if (clipDrag.isDragging && clipDrag.draggedClip && !isUpdating) {
      isUpdating = true
      
      try {
        // 只更新拖拽中的clip，完全跳过其他clip
        const draggedClipContainer = clipGraphicsCache.get(clipDrag.draggedClip.id)
        if (draggedClipContainer) {
          updateSingleClipPosition(clipDrag.draggedClip, draggedClipContainer)
        }
      } finally {
        isUpdating = false
      }
    }
  }, 300) // 增加到300ms延迟，进一步降低频率
}

// 简化的clip位置更新（确保波形跟随）
function updateSingleClipPosition(clip, clipContainer) {
  // 直接重新创建clip内容，确保波形跟随
  const trackIndex = tracks.value.findIndex(track => 
    track.clips.some(c => c.id === clip.id)
  )
  
  if (trackIndex !== -1) {
    const track = tracks.value[trackIndex]
    recreateClip(clip, track, trackIndex, clipContainer)
  }
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
  clipDrag.originalTrack = track
  clipDrag.targetTrack = track
  clipDrag.startX = event.data.global.x
  clipDrag.startY = event.data.global.y
  clipDrag.startTime = clip.startTime
  clipDrag.offsetX = 0
  clipDrag.lastUpdateTime = Date.now()
  clipDrag.pendingUpdate = false
  clipDrag.snapTime = 0
  clipDrag.snapToClip = null
  
  selectedClip.value = clip
  
  // 创建拖拽预览（半透明副本）
  if (!dragPreviewGraphics) {
    dragPreviewGraphics = new PIXI.Graphics()
    dragPreviewGraphics.alpha = 0.8
    tracksContainer.addChild(dragPreviewGraphics)
  }
  
  document.addEventListener('pointermove', handleClipDragOptimized)
  document.addEventListener('pointerup', stopClipDrag)
  
  console.log('开始拖拽片段:', clip.name, '原轨道:', track.name)
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

// 查找停靠点
function findSnapPoint(time, targetTrackIndex) {
  const snapDistance = 0.5 // 0.5秒内停靠
  let bestSnapTime = time
  let snapToClip = null
  
  if (targetTrackIndex >= 0 && targetTrackIndex < tracks.value.length) {
    const targetTrack = tracks.value[targetTrackIndex]
    
    // 检查与其他clip的停靠
    for (const clip of targetTrack.clips) {
      if (clip.id === clipDrag.draggedClip?.id) continue
      
      // 停靠到clip开始位置
      if (Math.abs(time - clip.startTime) < snapDistance) {
        bestSnapTime = clip.startTime
        snapToClip = clip
        break
      }
      
      // 停靠到clip结束位置
      const clipEndTime = clip.startTime + clip.duration
      if (Math.abs(time - clipEndTime) < snapDistance) {
        bestSnapTime = clipEndTime
        snapToClip = clip
        break
      }
    }
  }
  
  return { snapTime: Math.max(0, bestSnapTime), snapToClip }
}

// 检测目标轨道
function detectTargetTrack(mouseY) {
  const containerRect = pixiContainer.value?.getBoundingClientRect()
  if (!containerRect) return null
  
  const relativeY = mouseY - containerRect.top + scrollY.value
  const trackIndex = Math.floor(relativeY / (trackHeight * zoomY.value))
  
  return trackIndex >= 0 && trackIndex < tracks.value.length ? trackIndex : null
}

// 极简拖拽位置更新，专为多clip优化
function updateDragPosition(event) {
  const deltaX = event.clientX - clipDrag.startX
  
  // 计算新时间位置
  const newTime = clipDrag.startTime + (deltaX / (pixelsPerSecond * zoomX.value))
  
  // 更大的阈值，减少频繁更新
  const finalTime = Math.max(0, Math.round(newTime * 2) / 2) // 改为0.5秒对齐，减少计算
  
  if (clipDrag.draggedClip && Math.abs(clipDrag.draggedClip.startTime - finalTime) > 0.5) {
    clipDrag.draggedClip.startTime = finalTime
    
    // 暂时禁用轨道检测和预览更新，只在拖拽结束时处理
    // 这样可以大幅减少多clip场景下的计算负担
    
    // 极低频率的clip更新
    throttledUpdateClipPositions()
  }
}

// 简化的拖拽预览，减少渲染负担
function updateDragPreview(time, targetTrackIndex, snapToClip) {
  if (!dragPreviewGraphics) return
  
  // 限制预览更新频率
  const now = Date.now()
  if (clipDrag.lastPreviewUpdate && now - clipDrag.lastPreviewUpdate < 50) {
    return // 50ms内不重复更新预览
  }
  clipDrag.lastPreviewUpdate = now
  
  dragPreviewGraphics.clear()
  
  const newX = time * pixelsPerSecond * zoomX.value
  const clipWidth = clipDrag.draggedClip.duration * pixelsPerSecond * zoomX.value
  const clipHeight = trackHeight * zoomY.value - 20
  const color = parseInt(clipDrag.draggedClip.color.replace('#', ''), 16)
  
  // 计算Y位置
  let previewY = 10
  if (targetTrackIndex !== null) {
    previewY = targetTrackIndex * trackHeight * zoomY.value + 10
  } else {
    // 保持原轨道位置
    const originalTrackIndex = tracks.value.findIndex(track => 
      track.clips.some(clip => clip.id === clipDrag.draggedClip.id)
    )
    previewY = originalTrackIndex * trackHeight * zoomY.value + 10
  }
  
  // 简化的预览框（只绘制边框，减少填充）
  dragPreviewGraphics.lineStyle(3, color, snapToClip ? 1.0 : 0.6)
  dragPreviewGraphics.drawRoundedRect(newX, previewY, clipWidth, clipHeight, 6)
  
  // 简化的停靠指示器
  if (snapToClip) {
    dragPreviewGraphics.lineStyle(4, 0x00ff00, 1.0)
    dragPreviewGraphics.moveTo(newX - 2, previewY)
    dragPreviewGraphics.lineTo(newX - 2, previewY + clipHeight)
  }
}

// 检查时间范围重叠
function hasTimeOverlap(clip1, clip2) {
  const clip1End = clip1.startTime + clip1.duration
  const clip2End = clip2.startTime + clip2.duration
  
  return !(clip1End <= clip2.startTime || clip2End <= clip1.startTime)
}

// 查找轨道中最后一个clip的结束时间
function findLastClipEndTime(track) {
  if (!track.clips.length) return 0
  
  let maxEndTime = 0
  for (const clip of track.clips) {
    const endTime = clip.startTime + clip.duration
    if (endTime > maxEndTime) {
      maxEndTime = endTime
    }
  }
  return maxEndTime
}

function stopClipDrag() {
  if (clipDrag.isDragging) {
    const draggedClip = clipDrag.draggedClip
    const originalTrack = clipDrag.originalTrack
    
    console.log('停止拖拽片段:', draggedClip?.name, '新位置:', draggedClip?.startTime)
    
    // 简化处理：暂时只支持同轨道内移动，减少复杂度
    // 检查同轨道内冲突
    const hasConflict = originalTrack.clips.some(clip => 
      clip.id !== draggedClip.id && hasTimeOverlap(draggedClip, clip)
    )
    
    if (hasConflict) {
      // 有冲突，移动到轨道末尾
      const lastEndTime = findLastClipEndTime(originalTrack)
      draggedClip.startTime = lastEndTime
      console.log('检测到冲突，移动到轨道末尾:', lastEndTime)
    }
    
    // 清除拖拽预览
    if (dragPreviewGraphics) {
      dragPreviewGraphics.clear()
    }
    
    // 清除更新超时
    if (dragUpdateTimeout) {
      clearTimeout(dragUpdateTimeout)
      dragUpdateTimeout = null
    }
    
    // 最终重绘确保一致性
    createTracks()
  }
  
  // 重置拖拽状态
  clipDrag.isDragging = false
  clipDrag.draggedClip = null
  clipDrag.draggedClipGraphics = null
  clipDrag.originalTrack = null
  clipDrag.targetTrack = null
  clipDrag.pendingUpdate = false
  clipDrag.snapTime = 0
  clipDrag.snapToClip = null
  isUpdating = false // 重置更新标志
  
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
    const scrollSpeed = 80 // 增加滚动速度
    
    if (event.shiftKey) {
      // 水平滚动 - 修复计算逻辑
      const containerWidth = pixiContainer.value?.clientWidth || 800
      const contentWidth = maxDuration * pixelsPerSecond * zoomX.value
      const maxScrollX = Math.max(0, contentWidth - containerWidth)
      
      // 修复滚动方向和增量计算
      let newScrollX = scrollX.value
      if (event.deltaY > 0) {
        newScrollX += scrollSpeed // 向右滚动
      } else {
        newScrollX -= scrollSpeed // 向左滚动
      }
      
      scrollX.value = Math.max(0, Math.min(maxScrollX, newScrollX))
      console.log('水平滚动:', scrollX.value, '最大值:', maxScrollX, '内容宽度:', contentWidth)
    } else {
      // 垂直滚动
      const containerHeight = pixiContainer.value?.clientHeight || 400
      const contentHeight = tracks.value.length * trackHeight * zoomY.value
      const maxScrollY = Math.max(0, contentHeight - containerHeight)
      
      let newScrollY = scrollY.value
      if (event.deltaY > 0) {
        newScrollY += scrollSpeed // 向下滚动
      } else {
        newScrollY -= scrollSpeed // 向上滚动
      }
      
      scrollY.value = Math.max(0, Math.min(maxScrollY, newScrollY))
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
      
      // 修复鼠标拖拽滚动计算
      const newScrollX = scrollX.value - deltaX
      const newScrollY = scrollY.value - deltaY
      
      scrollX.value = Math.max(0, Math.min(maxScrollX, newScrollX))
      scrollY.value = Math.max(0, Math.min(maxScrollY, newScrollY))
      
      updateViewport()
      
      console.log('鼠标拖拽滚动 - deltaX:', deltaX, 'deltaY:', deltaY, 'scrollX:', scrollX.value, 'scrollY:', scrollY.value)
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
  
  // 确保轨道容器正确定位（水平和垂直滚动）
  if (tracksContainer) {
    tracksContainer.x = -scrollX.value
    tracksContainer.y = -scrollY.value
  }
  
  // 确保播放头跟随水平滚动
  updatePlayhead()
  drawTimeline()
  
  console.log('视口更新 - scrollX:', scrollX.value, 'scrollY:', scrollY.value)
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

// 键盘事件处理
function handleKeyDown(event) {
  const scrollSpeed = 100
  
  if (event.key === 'ArrowLeft') {
    // 向左滚动
    const containerWidth = pixiContainer.value?.clientWidth || 800
    const contentWidth = maxDuration * pixelsPerSecond * zoomX.value
    const maxScrollX = Math.max(0, contentWidth - containerWidth)
    
    scrollX.value = Math.max(0, scrollX.value - scrollSpeed)
    updateViewport()
    event.preventDefault()
  } else if (event.key === 'ArrowRight') {
    // 向右滚动
    const containerWidth = pixiContainer.value?.clientWidth || 800
    const contentWidth = maxDuration * pixelsPerSecond * zoomX.value
    const maxScrollX = Math.max(0, contentWidth - containerWidth)
    
    scrollX.value = Math.min(maxScrollX, scrollX.value + scrollSpeed)
    updateViewport()
    event.preventDefault()
  } else if (event.key === 'ArrowUp') {
    // 向上滚动
    const containerHeight = pixiContainer.value?.clientHeight || 400
    const contentHeight = tracks.value.length * trackHeight * zoomY.value
    const maxScrollY = Math.max(0, contentHeight - containerHeight)
    
    scrollY.value = Math.max(0, scrollY.value - scrollSpeed)
    updateViewport()
    event.preventDefault()
  } else if (event.key === 'ArrowDown') {
    // 向下滚动
    const containerHeight = pixiContainer.value?.clientHeight || 400
    const contentHeight = tracks.value.length * trackHeight * zoomY.value
    const maxScrollY = Math.max(0, contentHeight - containerHeight)
    
    scrollY.value = Math.min(maxScrollY, scrollY.value + scrollSpeed)
    updateViewport()
    event.preventDefault()
  }
}

// 生命周期
onMounted(async () => {
  await nextTick()
  await initPixi()
  window.addEventListener('resize', handleResize)
  window.addEventListener('keydown', handleKeyDown)
})

onUnmounted(() => {
  if (playInterval) {
    clearInterval(playInterval)
  }
  if (app) {
    window.removeEventListener('resize', handleResize)
    window.removeEventListener('keydown', handleKeyDown)
    app.destroy(true)
  }
  
  // 清理拖拽相关的事件监听器和缓存
  document.removeEventListener('pointermove', handleClipDragOptimized)
  document.removeEventListener('pointerup', stopClipDrag)
  clipGraphicsCache.clear()
  dragPreviewGraphics = null
  
  // 清理拖拽超时
  if (dragUpdateTimeout) {
    clearTimeout(dragUpdateTimeout)
  }
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
  height: calc(100% - 60px); /* 减去timeline-placeholder的高度 */
  min-height: 400px;
}

.track-info {
  position: absolute;
  width: 100%;
  padding: 8px 12px;
  box-sizing: border-box;
  border-bottom: 1px solid #333;
  display: flex;
  flex-direction: column;
  justify-content: center;
  cursor: default;
  transition: background-color 0.2s;
  background: rgba(0, 0, 0, 0.3);
}

.track-info:hover {
  background: rgba(255, 255, 255, 0.02);
}

.track-content {
  display: flex;
  flex-direction: column;
  gap: 6px;
  width: 100%;
}

.track-header {
  display: flex;
  flex-direction: column;
  gap: 2px;
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
  justify-content: space-between;
  align-items: center;
  gap: 8px;
  margin-top: 4px;
}

.track-buttons {
  display: flex;
  gap: 6px;
}

.track-controls .t-button {
  width: 24px;
  height: 24px;
  font-weight: bold;
  font-size: 11px;
  padding: 0;
  min-width: 24px;
}

.volume-control {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 10px;
}

.volume-label {
  color: #999;
  font-size: 10px;
  min-width: 20px;
  text-align: right;
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