<template>
  <div class="pixi-track-editor">
    <!-- 控制面板 -->
    <div class="controls-panel">
      <div class="panel-left">
        <h3>🎵 轨道编辑器</h3>
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
        </div>
      </div>
    </div>

    <!-- PixiJS 渲染区域 -->
    <div 
      ref="pixiContainer" 
      class="pixi-container"
      tabindex="0"
      @wheel="handleWheel"
      @mousedown="handleMouseDown"
      @mousemove="handleMouseMove"
      @mouseup="handleMouseUp"
      @mouseleave="handleMouseUp"
      @focus="handleFocus"
    >
      <!-- 滚动条 -->
      <div class="scrollbars">
        <div 
          class="horizontal-scrollbar"
          @mousedown="handleHorizontalScrollClick"
        >
          <div 
            class="scroll-thumb"
            :style="horizontalThumbStyle"
            @mousedown.stop="startHorizontalScrollDrag"
          ></div>
        </div>
        <div 
          class="vertical-scrollbar"
          @mousedown="handleVerticalScrollClick"
        >
          <div 
            class="scroll-thumb"
            :style="verticalThumbStyle"
            @mousedown.stop="startVerticalScrollDrag"
          ></div>
        </div>
      </div>
    </div>

    <!-- 轨道信息侧边栏 -->
    <div class="track-sidebar">
      <div 
        v-for="(track, index) in sortedTracks" 
        :key="track.id"
        class="track-info"
        :class="{ 'track-dragging': trackDrag.draggedTrackId === track.id }"
        :style="{ 
          top: (index * trackHeight * zoomY) + 'px',
          height: (trackHeight * zoomY) + 'px'
        }"
        @mousedown="handleTrackMouseDown($event, track, index)"
        draggable="false"
      >
        <!-- 拖拽手柄 -->
        <div class="drag-handle">
          <div class="drag-dots">
            <div class="dot"></div>
            <div class="dot"></div>
            <div class="dot"></div>
            <div class="dot"></div>
          </div>
        </div>
        
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
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick, computed, reactive } from 'vue'
import * as PIXI from 'pixi.js'

// 响应式状态
const pixiContainer = ref(null)
const isPlaying = ref(false)
const currentTime = ref(0)
const zoomX = ref(1)
const zoomY = ref(1)
const scrollX = ref(0)
const scrollY = ref(0)

// 视口和布局常量
const trackHeight = 120
const timelineHeight = 60
const pixelsPerSecond = 100
const maxDuration = 300 // 5分钟最大时长

// 鼠标和滚动状态
const mouse = reactive({
  isDown: false,
  lastX: 0,
  lastY: 0,
  isDragging: false
})

// 滚动条拖拽状态
const scrollbarDrag = reactive({
  isHorizontalDragging: false,
  isVerticalDragging: false,
  startX: 0,
  startY: 0,
  startScrollX: 0,
  startScrollY: 0
})

// PixiJS 相关变量
let app = null
let mainContainer = null
let tracksContainer = null
let timelineContainer = null
let playheadLine = null

// 性能优化相关
let viewportBounds = { left: 0, right: 0, top: 0, bottom: 0 }
let renderCache = new Map()
let lastViewport = { x: 0, y: 0, zoomX: 1, zoomY: 1 }
let needsRedraw = true

// 轨道数据结构
const tracks = ref([
  {
    id: 1,
    name: '华丽主旋律',
    type: 'LEAD',
    color: '#10b981',
    volume: 85,
    isSolo: false,
    isMuted: false,
    order: 0,
    clips: [
      {
        id: 'clip_1_1',
        name: '主旋律_1',
        startTime: 0,
        duration: 15,
        color: '#10b981',
        waveformData: []
      },
      {
        id: 'clip_1_2', 
        name: '主旋律_2',
        startTime: 20,
        duration: 12,
        color: '#059669',
        waveformData: []
      }
    ]
  },
  {
    id: 2,
    name: '节奏鼓点',
    type: 'PERC',
    color: '#f59e0b',
    volume: 95,
    isSolo: false,
    isMuted: false,
    order: 1,
    clips: [
      {
        id: 'clip_2_1',
        name: '鼓点_1',
        startTime: 0,
        duration: 8,
        color: '#f59e0b',
        waveformData: []
      },
      {
        id: 'clip_2_2',
        name: '鼓点_2', 
        startTime: 12,
        duration: 10,
        color: '#d97706',
        waveformData: []
      },
      {
        id: 'clip_2_3',
        name: '鼓点_3',
        startTime: 25,
        duration: 8,
        color: '#f59e0b',
        waveformData: []
      }
    ]
  },
  {
    id: 3,
    name: '管乐和弦',
    type: 'WIND',
    color: '#8b5cf6',
    volume: 90,
    isSolo: false,
    isMuted: false,
    order: 2,
    clips: [
      {
        id: 'clip_3_1',
        name: '和弦_1',
        startTime: 5,
        duration: 18,
        color: '#8b5cf6',
        waveformData: []
      }
    ]
  },
  {
    id: 4,
    name: '深邃贝斯',
    type: 'BASS',
    color: '#3b82f6',
    volume: 70,
    isSolo: false,
    isMuted: false,
    order: 3,
    clips: [
      {
        id: 'clip_4_1',
        name: '贝斯_1',
        startTime: 0,
        duration: 25,
        color: '#3b82f6',
        waveformData: []
      }
    ]
  },
  {
    id: 5,
    name: '天籁和声',
    type: 'PAD',
    color: '#ef4444',
    volume: 55,
    isSolo: false,
    isMuted: false,
    order: 4,
    clips: [
      {
        id: 'clip_5_1',
        name: '和声_1',
        startTime: 10,
        duration: 20,
        color: '#ef4444',
        waveformData: []
      },
      {
        id: 'clip_5_2',
        name: '和声_2',
        startTime: 35,
        duration: 15,
        color: '#dc2626',
        waveformData: []
      }
    ]
  }
])

// 片段拖拽状态
const clipDrag = reactive({
  isDragging: false,
  draggedClip: null,
  draggedTrackId: null,
  startX: 0,
  startTime: 0,
  offsetX: 0
})

// 轨道拖拽状态
const trackDrag = reactive({
  isDragging: false,
  draggedTrackId: null,
  startY: 0,
  startOrder: 0
})

// 计算属性
const viewportStartTime = computed(() => scrollX.value / (pixelsPerSecond * zoomX.value))
const viewportEndTime = computed(() => {
  const containerWidth = pixiContainer.value?.clientWidth || 800
  return viewportStartTime.value + (containerWidth / (pixelsPerSecond * zoomX.value))
})

// 按order排序的轨道
const sortedTracks = computed(() => {
  return [...tracks.value].sort((a, b) => a.order - b.order)
})

const horizontalThumbStyle = computed(() => {
  const containerWidth = pixiContainer.value?.clientWidth || 800
  const contentWidth = maxDuration * pixelsPerSecond * zoomX.value
  
  if (contentWidth <= containerWidth) {
    // 内容不超出容器时，隐藏滚动条
    return { width: '0px', left: '0px' }
  }
  
  const thumbWidth = Math.max(20, (containerWidth / contentWidth) * containerWidth)
  const maxScrollX = contentWidth - containerWidth
  const thumbLeft = maxScrollX > 0 ? (scrollX.value / maxScrollX) * (containerWidth - thumbWidth) : 0
  
  return {
    width: thumbWidth + 'px',
    left: thumbLeft + 'px'
  }
})

const verticalThumbStyle = computed(() => {
  const containerHeight = (pixiContainer.value?.clientHeight || 600) - timelineHeight
  const contentHeight = tracks.value.length * trackHeight * zoomY.value
  
  if (contentHeight <= containerHeight) {
    // 内容不超出容器时，隐藏滚动条
    return { height: '0px', top: '0px' }
  }
  
  const thumbHeight = Math.max(20, (containerHeight / contentHeight) * containerHeight)
  const maxScrollY = contentHeight - containerHeight
  const thumbTop = maxScrollY > 0 ? (scrollY.value / maxScrollY) * (containerHeight - thumbHeight) : 0
  
  return {
    height: thumbHeight + 'px',
    top: thumbTop + 'px'
  }
})

// 为clip生成波形数据
function generateClipWaveformData(clip, trackType) {
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
      case 'WIND':
        amplitude = Math.sin(time * 3 + Math.sin((time + clip.startTime) * 0.5)) * 0.7
        break
      case 'PAD':
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

// 检测OffscreenCanvas支持
function supportsOffscreenCanvas() {
  try {
    return typeof OffscreenCanvas !== 'undefined' && 
           typeof OffscreenCanvas.prototype.getContext !== 'undefined' &&
           typeof OffscreenCanvas.prototype.transferToImageBitmap !== 'undefined'
  } catch (e) {
    return false
  }
}

// 初始化 PixiJS
async function initPixi() {
  const container = pixiContainer.value
  if (!container) return

  // 基础配置
  const width = container.clientWidth || 800
  const height = container.clientHeight || 600

  try {
    // 先尝试标准方式创建PixiJS应用
    console.log('🎯 初始化PixiJS应用...')
    
    const pixiOptions = {
      width,
      height,
      backgroundColor: 0x1a1a1a,
      antialias: true,
      resolution: window.devicePixelRatio || 1,
      autoDensity: true,
      powerPreference: 'high-performance'
    }

    // 优先使用标准Canvas，确保稳定性
    app = new PIXI.Application(pixiOptions)
    container.appendChild(app.view)
    console.log('✅ 标准Canvas初始化成功')

    // 如果标准方式成功，可以尝试OffscreenCanvas优化（可选）
    if (supportsOffscreenCanvas() && false) { // 暂时禁用OffscreenCanvas
      try {
        console.log('🚀 尝试OffscreenCanvas优化...')
        
        // 创建OffscreenCanvas作为渲染目标
        const offscreenCanvas = new OffscreenCanvas(width, height)
        const offscreenCtx = offscreenCanvas.getContext('2d')
        
        if (offscreenCtx) {
          // 成功创建OffscreenCanvas上下文
          console.log('✅ OffscreenCanvas上下文创建成功')
          
          // 这里可以添加OffscreenCanvas的额外优化逻辑
          // 但保持主渲染路径使用标准Canvas确保稳定性
        }
      } catch (offscreenError) {
        console.warn('⚠️ OffscreenCanvas优化失败，继续使用标准Canvas:', offscreenError)
      }
    }

  } catch (error) {
    console.error('❌ PixiJS初始化失败:', error)
    
    // 最后的降级方案 - 使用最基础的配置
    try {
      console.log('🔄 尝试基础配置降级...')
      app = new PIXI.Application({
        width: width,
        height: height,
        backgroundColor: 0x1a1a1a,
        antialias: false, // 禁用抗锯齿
        resolution: 1, // 固定分辨率
        autoDensity: false
      })
      container.appendChild(app.view)
      console.log('✅ 基础配置初始化成功')
    } catch (fallbackError) {
      console.error('💥 所有初始化方案都失败了:', fallbackError)
      
      // 创建错误提示
      const errorDiv = document.createElement('div')
      errorDiv.style.cssText = `
        padding: 20px;
        background: #ff4444;
        color: white;
        border-radius: 8px;
        text-align: center;
        font-family: Arial, sans-serif;
      `
      errorDiv.innerHTML = `
        <h3>渲染引擎初始化失败</h3>
        <p>请尝试刷新页面或使用其他浏览器</p>
        <p>错误信息: ${fallbackError.message}</p>
      `
      container.appendChild(errorDiv)
      return
    }
  }

  // 确保app正确初始化
  if (!app || !app.stage) {
    console.error('❌ PixiJS应用未正确初始化')
    return
  }

  try {
    // 创建主容器
    mainContainer = new PIXI.Container()
    app.stage.addChild(mainContainer)

    // 创建时间线容器
    timelineContainer = new PIXI.Container()
    timelineContainer.y = 0
    mainContainer.addChild(timelineContainer)

    // 创建轨道容器
    tracksContainer = new PIXI.Container()
    tracksContainer.y = timelineHeight
    mainContainer.addChild(tracksContainer)

    // 创建播放头
    playheadLine = new PIXI.Graphics()
    playheadLine.lineStyle(2, 0xff4444)
    playheadLine.moveTo(0, 0)
    playheadLine.lineTo(0, height)
    mainContainer.addChild(playheadLine)

    // 初始化轨道数据
    initializeTracks()
    
    // 创建时间线
    createTimeline()
    
    // 创建轨道
    createTracks()
    
    // 开始渲染循环
    startRenderLoop()
    
    console.log('✅ PixiJS场景初始化完成')
  } catch (sceneError) {
    console.error('❌ 场景初始化失败:', sceneError)
  }
}

// 初始化轨道数据
function initializeTracks() {
  tracks.value.forEach(track => {
    track.clips.forEach(clip => {
      if (!clip.waveformData.length) {
        clip.waveformData = generateClipWaveformData(clip, track.type)
      }
    })
  })
}

// 更新视口边界
function updateViewportBounds() {
  const containerWidth = pixiContainer.value?.clientWidth || 800
  const containerHeight = (pixiContainer.value?.clientHeight || 600) - timelineHeight
  
  // 计算可视时间范围（增加缓冲区）
  const bufferTime = 2 / zoomX.value // 2秒缓冲
  const startTime = Math.max(0, viewportStartTime.value - bufferTime)
  const endTime = Math.min(maxDuration, viewportEndTime.value + bufferTime)
  
  // 计算可视轨道范围
  const startTrack = Math.max(0, Math.floor(scrollY.value / (trackHeight * zoomY.value)))
  const endTrack = Math.min(tracks.value.length - 1, 
    Math.ceil((scrollY.value + containerHeight) / (trackHeight * zoomY.value)))
  
  viewportBounds = {
    left: startTime * pixelsPerSecond * zoomX.value,
    right: endTime * pixelsPerSecond * zoomX.value,
    top: startTrack,
    bottom: endTrack,
    startTime,
    endTime,
    startTrack,
    endTrack
  }
}

// 创建优化的时间线
function createTimeline() {
  timelineContainer.removeChildren()
  updateViewportBounds()
  
  // 只渲染可视范围的时间线背景
  const timelineBackground = new PIXI.Graphics()
  timelineBackground.beginFill(0x2a2a2a)
  timelineBackground.drawRect(
    viewportBounds.left, 0, 
    viewportBounds.right - viewportBounds.left, timelineHeight
  )
  timelineBackground.endFill()
  timelineContainer.addChild(timelineBackground)
  
  // 时间刻度 - 只渲染可视范围
  const timeStep = Math.max(1, Math.floor(10 / zoomX.value))
  const startTick = Math.floor(viewportBounds.startTime / timeStep) * timeStep
  const endTick = Math.ceil(viewportBounds.endTime / timeStep) * timeStep
  
  for (let t = startTick; t <= endTick; t += timeStep) {
    const x = t * pixelsPerSecond * zoomX.value
    
    // 跳过不在可视范围内的刻度
    if (x < viewportBounds.left || x > viewportBounds.right) continue
    
    // 主要刻度线
    const majorTick = new PIXI.Graphics()
    majorTick.lineStyle(1, 0x666666)
    majorTick.moveTo(x, timelineHeight - 20)
    majorTick.lineTo(x, timelineHeight)
    timelineContainer.addChild(majorTick)
    
    // 时间标签 - 减少密度以提高性能
    if (t % Math.max(timeStep, 5) === 0) {
      const timeText = new PIXI.Text(formatTime(t), {
        fontSize: 12,
        fill: 0xffffff
      })
      timeText.x = x + 2
      timeText.y = timelineHeight - 35
      timelineContainer.addChild(timeText)
    }
    
    // 次要刻度线 - 只在高缩放时显示
    if (zoomX.value > 1) {
      for (let subT = 0.2; subT < timeStep && subT < 1; subT += 0.2) {
        const subX = (t + subT) * pixelsPerSecond * zoomX.value
        if (subX >= viewportBounds.left && subX <= viewportBounds.right) {
          const minorTick = new PIXI.Graphics()
          minorTick.lineStyle(1, 0x444444)
          minorTick.moveTo(subX, timelineHeight - 10)
          minorTick.lineTo(subX, timelineHeight)
          timelineContainer.addChild(minorTick)
        }
      }
    }
  }
}

// 优化的轨道创建 - 支持多个clips
function createTracks() {
  tracksContainer.removeChildren()
  updateViewportBounds()
  
  // 使用排序后的轨道
  const tracks = sortedTracks.value
  
  // 只渲染可视范围内的轨道
  for (let index = viewportBounds.startTrack; index <= viewportBounds.endTrack; index++) {
    if (index >= tracks.length) break
    
    const track = tracks[index]
    const trackContainer = new PIXI.Container()
    trackContainer.y = index * trackHeight * zoomY.value
    
    // 轨道背景 - 只渲染可视宽度
    const trackBg = new PIXI.Graphics()
    trackBg.beginFill(index % 2 ? 0x1e1e1e : 0x252525)
    trackBg.drawRect(
      viewportBounds.left, 0, 
      viewportBounds.right - viewportBounds.left, 
      trackHeight * zoomY.value
    )
    trackBg.endFill()
    trackBg.interactive = true
    trackBg.trackId = track.id
    trackContainer.addChild(trackBg)
    
    // 轨道分割线 - 只渲染可视宽度
    const separator = new PIXI.Graphics()
    separator.lineStyle(1, 0x333333)
    separator.moveTo(viewportBounds.left, trackHeight * zoomY.value)
    separator.lineTo(viewportBounds.right, trackHeight * zoomY.value)
    trackContainer.addChild(separator)
    
    // 渲染轨道中的所有clips
    track.clips.forEach(clip => {
      const clipStartX = clip.startTime * pixelsPerSecond * zoomX.value
      const clipEndX = (clip.startTime + clip.duration) * pixelsPerSecond * zoomX.value
      
      // 只渲染在可视范围内的clips
      if (clipEndX >= viewportBounds.left && clipStartX <= viewportBounds.right) {
        const clipContainer = createClip(clip, track, index)
        if (clipContainer) {
          trackContainer.addChild(clipContainer)
        }
      }
    })
    
    tracksContainer.addChild(trackContainer)
  }
}

// 创建clip容器和波形
function createClip(clip, track, trackIndex) {
  const clipContainer = new PIXI.Container()
  
  const clipWidth = clip.duration * pixelsPerSecond * zoomX.value
  const clipHeight = trackHeight * zoomY.value
  const clipX = clip.startTime * pixelsPerSecond * zoomX.value
  
  // Clip背景
  const clipBg = new PIXI.Graphics()
  clipBg.beginFill(PIXI.utils.hex2rgb(clip.color), 0.3)
  clipBg.drawRoundedRect(clipX, 10, clipWidth, clipHeight - 20, 6)
  clipBg.endFill()
  
  // Clip边框
  clipBg.lineStyle(2, PIXI.utils.hex2rgb(clip.color), track.isMuted ? 0.4 : 0.8)
  clipBg.drawRoundedRect(clipX, 10, clipWidth, clipHeight - 20, 6)
  
  // 设置交互
  clipBg.interactive = true
  clipBg.buttonMode = true
  clipBg.clipId = clip.id
  clipBg.trackId = track.id
  
  // 添加鼠标事件
  clipBg.on('pointerdown', (event) => handleClipMouseDown(event, clip, track))
  clipBg.on('pointerover', () => clipBg.alpha = 0.8)
  clipBg.on('pointerout', () => clipBg.alpha = 1.0)
  
  clipContainer.addChild(clipBg)
  
  // 创建波形
  if (clip.waveformData && clip.waveformData.length > 0) {
    const waveform = createClipWaveform(clip, track)
    if (waveform) {
      waveform.x = clipX
      clipContainer.addChild(waveform)
    }
  }
  
  // Clip标题
  const clipText = new PIXI.Text(clip.name, {
    fontSize: 12,
    fill: 0xffffff,
    fontWeight: 'bold'
  })
  clipText.x = clipX + 8
  clipText.y = 15
  clipContainer.addChild(clipText)
  
  // Clip时间信息
  const timeText = new PIXI.Text(`${clip.duration.toFixed(1)}s`, {
    fontSize: 10,
    fill: 0xcccccc
  })
  timeText.x = clipX + 8
  timeText.y = clipHeight - 25
  clipContainer.addChild(timeText)
  
  return clipContainer
}

// 创建clip内的波形
function createClipWaveform(clip, track) {
  const waveformContainer = new PIXI.Container()
  const waveformData = clip.waveformData
  
  if (!waveformData || waveformData.length === 0) return null
  
  const clipWidth = clip.duration * pixelsPerSecond * zoomX.value
  const clipHeight = trackHeight * zoomY.value
  const pointWidth = clipWidth / waveformData.length
  
  const color = PIXI.utils.hex2rgb(clip.color)
  const alpha = track.isMuted ? 0.3 : 0.6
  
  // 创建波形路径
  const waveform = new PIXI.Graphics()
  waveform.alpha = alpha
  
  const baselineY = clipHeight / 2
  const amplitudeScale = (clipHeight - 40) / 4 // 留出空间给文字
  
  waveform.beginFill(color, 0.6)
  waveform.moveTo(0, baselineY)
  
  // 上半部分路径
  for (let i = 0; i < waveformData.length; i++) {
    const x = i * pointWidth
    const amplitude = waveformData[i] * amplitudeScale
    waveform.lineTo(x, baselineY - amplitude)
  }
  
  // 下半部分路径（镜像）
  for (let i = waveformData.length - 1; i >= 0; i--) {
    const x = i * pointWidth
    const amplitude = waveformData[i] * amplitudeScale
    waveform.lineTo(x, baselineY + amplitude)
  }
  
  waveform.closePath()
  waveform.endFill()
  
  waveformContainer.addChild(waveform)
  return waveformContainer
}

// 优化的波形创建 - 支持缓存和可视区域渲染
function createOptimizedWaveform(track, trackIndex) {
  const cacheKey = `${track.id}-${zoomX.value.toFixed(2)}-${zoomY.value.toFixed(2)}-${viewportBounds.startTime.toFixed(2)}-${viewportBounds.endTime.toFixed(2)}`
  
  // 检查缓存
  if (renderCache.has(cacheKey)) {
    const cached = renderCache.get(cacheKey)
    cached.alpha = track.isMuted ? 0.3 : 0.8
    return cached
  }
  
  const waveformContainer = new PIXI.Container()
  const waveformData = track.waveformData
  
  if (waveformData.length === 0) return null
  
  const totalDuration = track.duration
  const pointsPerSecond = waveformData.length / totalDuration
  
  // 计算可视范围内的数据点
  const startDataIndex = Math.max(0, Math.floor(viewportBounds.startTime * pointsPerSecond))
  const endDataIndex = Math.min(waveformData.length - 1, Math.ceil(viewportBounds.endTime * pointsPerSecond))
  
  const color = PIXI.utils.hex2rgb(track.color)
  const alpha = track.isMuted ? 0.3 : 0.8
  
  // 创建优化的波形路径
  const waveform = new PIXI.Graphics()
  waveform.alpha = alpha
  
  // 动态采样率 - 根据缩放级别调整
  const sampleRate = Math.max(1, Math.floor(1 / zoomX.value))
  const baselineY = trackHeight * zoomY.value / 2
  const amplitudeScale = trackHeight * zoomY.value / 4
  
  waveform.beginFill(color, 0.8)
  waveform.moveTo(viewportBounds.left, baselineY)
  
  // 上半部分路径
  for (let i = startDataIndex; i <= endDataIndex; i += sampleRate) {
    if (i >= waveformData.length) break
    
    const time = i / pointsPerSecond
    const x = time * pixelsPerSecond * zoomX.value
    const amplitude = waveformData[i] * amplitudeScale
    
    waveform.lineTo(x, baselineY - amplitude)
  }
  
  // 下半部分路径（镜像）
  for (let i = endDataIndex; i >= startDataIndex; i -= sampleRate) {
    if (i < 0) break
    
    const time = i / pointsPerSecond
    const x = time * pixelsPerSecond * zoomX.value
    const amplitude = waveformData[i] * amplitudeScale
    
    waveform.lineTo(x, baselineY + amplitude)
  }
  
  waveform.closePath()
  waveform.endFill()
  
  waveformContainer.addChild(waveform)
  
  // 缓存结果（限制缓存大小）
  if (renderCache.size > 50) {
    const firstKey = renderCache.keys().next().value
    renderCache.delete(firstKey)
  }
  renderCache.set(cacheKey, waveformContainer)
  
  return waveformContainer
}

// 原始波形创建函数（备用）
function createWaveform(track) {
  const waveformContainer = new PIXI.Container()
  const waveformData = track.waveformData
  const waveWidth = track.duration * pixelsPerSecond * zoomX.value
  const pointWidth = waveWidth / waveformData.length
  
  const color = PIXI.utils.hex2rgb(track.color)
  const alpha = track.isMuted ? 0.3 : 0.8
  
  // 创建波形路径
  const waveform = new PIXI.Graphics()
  waveform.alpha = alpha
  
  // 绘制波形上半部分
  waveform.beginFill(color, 0.8)
  waveform.moveTo(0, trackHeight * zoomY.value / 2)
  
  for (let i = 0; i < waveformData.length; i++) {
    const x = i * pointWidth
    const amplitude = waveformData[i] * (trackHeight * zoomY.value / 4)
    waveform.lineTo(x, (trackHeight * zoomY.value / 2) - amplitude)
  }
  
  for (let i = waveformData.length - 1; i >= 0; i--) {
    const x = i * pointWidth
    const amplitude = waveformData[i] * (trackHeight * zoomY.value / 4)
    waveform.lineTo(x, (trackHeight * zoomY.value / 2) + amplitude)
  }
  
  waveform.closePath()
  waveform.endFill()
  
  waveformContainer.addChild(waveform)
  return waveformContainer
}

// 优化的缩放更新
function updateZoom() {
  // 检查是否真的需要重新渲染
  const threshold = 0.01
  if (Math.abs(lastViewport.zoomX - zoomX.value) < threshold && 
      Math.abs(lastViewport.zoomY - zoomY.value) < threshold) {
    return
  }
  
  // 清理缓存
  renderCache.clear()
  
  // 延迟渲染以避免频繁更新
  clearTimeout(updateZoom.timeoutId)
  updateZoom.timeoutId = setTimeout(() => {
    createTimeline()
    createTracks()
    updatePlayhead()
    
    lastViewport.zoomX = zoomX.value
    lastViewport.zoomY = zoomY.value
    needsRedraw = true
  }, 16) // 约60fps
}

// 更新播放头位置
function updatePlayhead() {
  if (playheadLine) {
    playheadLine.x = currentTime.value * pixelsPerSecond * zoomX.value - scrollX.value
  }
}

// 优化的视口更新
function updateViewport() {
  // 检查滚动变化
  const threshold = 5 // 像素
  if (Math.abs(lastViewport.x - scrollX.value) < threshold && 
      Math.abs(lastViewport.y - scrollY.value) < threshold) {
    return
  }
  
  if (mainContainer) {
    mainContainer.x = -scrollX.value
    mainContainer.y = -scrollY.value
  }
  
  // 延迟重新渲染可视区域
  clearTimeout(updateViewport.timeoutId)
  updateViewport.timeoutId = setTimeout(() => {
    // 只有滚动距离足够大时才重新渲染轨道
    if (Math.abs(lastViewport.x - scrollX.value) > 100 || 
        Math.abs(lastViewport.y - scrollY.value) > trackHeight * zoomY.value) {
      createTracks()
      createTimeline()
    }
    
    lastViewport.x = scrollX.value
    lastViewport.y = scrollY.value
    needsRedraw = true
  }, 33) // 约30fps，减少渲染频率
  
  updatePlayhead()
}

// 优化的渲染循环
function startRenderLoop() {
  let lastTime = 0
  
  function animate(currentTimeStamp) {
    const deltaTime = currentTimeStamp - lastTime
    
    // 限制帧率以提升性能
    if (deltaTime >= 16.67) { // 约60fps
      if (isPlaying.value) {
        currentTime.value += deltaTime / 1000 // 转换为秒
        
        // 自动滚动跟随播放头
        const playheadX = currentTime.value * pixelsPerSecond * zoomX.value
        const containerWidth = pixiContainer.value?.clientWidth || 800
        
        if (playheadX - scrollX.value > containerWidth * 0.8) {
          scrollX.value = playheadX - containerWidth * 0.2
          updateViewport()
        }
        
        updatePlayhead()
        needsRedraw = true
      }
      
      // 重置重绘标志
      if (needsRedraw) {
        needsRedraw = false
      }
      
      lastTime = currentTimeStamp
    }
    
    requestAnimationFrame(animate)
  }
  animate(0)
}

// 事件处理
function togglePlayback() {
  isPlaying.value = !isPlaying.value
}

function handleWheel(event) {
  event.preventDefault()
  
  if (event.ctrlKey || event.metaKey) {
    // 缩放操作
    const zoomFactor = event.deltaY > 0 ? 0.9 : 1.1
    if (event.shiftKey) {
      // Ctrl+Shift+滚轮：垂直缩放
      zoomY.value = Math.max(0.5, Math.min(3, zoomY.value * zoomFactor))
    } else {
      // Ctrl+滚轮：水平缩放
      zoomX.value = Math.max(0.1, Math.min(5, zoomX.value * zoomFactor))
    }
    updateZoom()
  } else {
    // 滚动操作
    const scrollSpeed = 50 // 滚动速度
    
    if (event.shiftKey) {
      // Shift+滚轮：水平滚动
      const maxScrollX = Math.max(0, maxDuration * pixelsPerSecond * zoomX.value - (pixiContainer.value?.clientWidth || 800))
      scrollX.value = Math.max(0, Math.min(maxScrollX, scrollX.value + event.deltaY * scrollSpeed / 100))
    } else {
      // 普通滚轮：垂直滚动
      const containerHeight = (pixiContainer.value?.clientHeight || 600) - timelineHeight
      const maxScrollY = Math.max(0, tracks.value.length * trackHeight * zoomY.value - containerHeight)
      scrollY.value = Math.max(0, Math.min(maxScrollY, scrollY.value + event.deltaY * scrollSpeed / 100))
    }
    updateViewport()
  }
}

function handleMouseDown(event) {
  // 如果正在拖拽clip或轨道，不处理容器拖拽
  if (clipDrag.isDragging || trackDrag.isDragging) {
    return
  }
  
  const rect = pixiContainer.value.getBoundingClientRect()
  const localY = event.clientY - rect.top
  
  mouse.isDown = true
  mouse.lastX = event.clientX
  mouse.lastY = event.clientY
  mouse.isDragging = false
  
  // 如果点击在时间线区域，设置播放位置
  if (localY < timelineHeight) {
    const localX = event.clientX - rect.left + scrollX.value
    const clickTime = localX / (pixelsPerSecond * zoomX.value)
    currentTime.value = Math.max(0, Math.min(maxDuration, clickTime))
    updatePlayhead()
  }
  
  // 设置鼠标样式
  pixiContainer.value.style.cursor = 'grabbing'
}

function handleMouseMove(event) {
  // 如果正在拖拽clip或轨道，不处理容器拖拽
  if (clipDrag.isDragging || trackDrag.isDragging) {
    return
  }
  
  if (mouse.isDown) {
    const deltaX = event.clientX - mouse.lastX
    const deltaY = event.clientY - mouse.lastY
    
    // 检测是否开始拖拽（防止误触）
    if (!mouse.isDragging && (Math.abs(deltaX) > 3 || Math.abs(deltaY) > 3)) {
      mouse.isDragging = true
    }
    
    if (mouse.isDragging) {
      // 计算滚动边界
      const containerWidth = pixiContainer.value?.clientWidth || 800
      const containerHeight = (pixiContainer.value?.clientHeight || 600) - timelineHeight
      const maxScrollX = Math.max(0, maxDuration * pixelsPerSecond * zoomX.value - containerWidth)
      const maxScrollY = Math.max(0, Math.min(tracks.value.length * trackHeight * zoomY.value - containerHeight, tracks.value.length * trackHeight * zoomY.value))
      
      // 应用滚动（反向移动）
      scrollX.value = Math.max(0, Math.min(maxScrollX, scrollX.value - deltaX))
      scrollY.value = Math.max(0, Math.min(maxScrollY, scrollY.value - deltaY))
      
      updateViewport()
    }
    
    mouse.lastX = event.clientX
    mouse.lastY = event.clientY
  }
}

function handleMouseUp() {
  // 只有在非clip/轨道拖拽时才处理容器鼠标释放
  if (!clipDrag.isDragging && !trackDrag.isDragging) {
    mouse.isDown = false
    mouse.isDragging = false
    
    // 恢复鼠标样式
    if (pixiContainer.value) {
      pixiContainer.value.style.cursor = 'grab'
    }
  }
  
  // 停止滚动条拖拽
  scrollbarDrag.isHorizontalDragging = false
  scrollbarDrag.isVerticalDragging = false
}

function handleFocus() {
  // 当容器获得焦点时，确保可以接收键盘事件
  console.log('🎯 编辑器获得焦点，可以使用键盘快捷键')
}

// 滚动条相关事件处理
function handleHorizontalScrollClick(event) {
  const rect = event.currentTarget.getBoundingClientRect()
  const clickX = event.clientX - rect.left
  const containerWidth = pixiContainer.value?.clientWidth || 800
  const contentWidth = maxDuration * pixelsPerSecond * zoomX.value
  const maxScrollX = Math.max(0, contentWidth - containerWidth)
  
  if (maxScrollX <= 0) return // 无需滚动
  
  // 计算点击位置对应的滚动值
  const scrollRatio = clickX / rect.width
  scrollX.value = Math.max(0, Math.min(maxScrollX, scrollRatio * maxScrollX))
  updateViewport()
}

function handleVerticalScrollClick(event) {
  const rect = event.currentTarget.getBoundingClientRect()
  const clickY = event.clientY - rect.top
  const containerHeight = (pixiContainer.value?.clientHeight || 600) - timelineHeight
  const contentHeight = tracks.value.length * trackHeight * zoomY.value
  const maxScrollY = Math.max(0, contentHeight - containerHeight)
  
  if (maxScrollY <= 0) return // 无需滚动
  
  // 计算点击位置对应的滚动值
  const scrollRatio = clickY / rect.height
  scrollY.value = Math.max(0, Math.min(maxScrollY, scrollRatio * maxScrollY))
  updateViewport()
}

function startHorizontalScrollDrag(event) {
  scrollbarDrag.isHorizontalDragging = true
  scrollbarDrag.startX = event.clientX
  scrollbarDrag.startScrollX = scrollX.value
  
  // 添加全局鼠标事件监听
  document.addEventListener('mousemove', handleScrollbarDrag)
  document.addEventListener('mouseup', stopScrollbarDrag)
}

function startVerticalScrollDrag(event) {
  scrollbarDrag.isVerticalDragging = true
  scrollbarDrag.startY = event.clientY
  scrollbarDrag.startScrollY = scrollY.value
  
  // 添加全局鼠标事件监听
  document.addEventListener('mousemove', handleScrollbarDrag)
  document.addEventListener('mouseup', stopScrollbarDrag)
}

function handleScrollbarDrag(event) {
  if (scrollbarDrag.isHorizontalDragging) {
    const deltaX = event.clientX - scrollbarDrag.startX
    const containerWidth = pixiContainer.value?.clientWidth || 800
    const contentWidth = maxDuration * pixelsPerSecond * zoomX.value
    const maxScrollX = Math.max(0, contentWidth - containerWidth)
    
    if (maxScrollX > 0) {
      // 根据滚动条的移动比例计算滚动值
      const scrollRatio = deltaX / containerWidth
      const newScrollX = scrollbarDrag.startScrollX + scrollRatio * maxScrollX
      scrollX.value = Math.max(0, Math.min(maxScrollX, newScrollX))
      updateViewport()
    }
  }
  
  if (scrollbarDrag.isVerticalDragging) {
    const deltaY = event.clientY - scrollbarDrag.startY
    const containerHeight = (pixiContainer.value?.clientHeight || 600) - timelineHeight
    const contentHeight = tracks.value.length * trackHeight * zoomY.value
    const maxScrollY = Math.max(0, contentHeight - containerHeight)
    
    if (maxScrollY > 0) {
      // 根据滚动条的移动比例计算滚动值
      const scrollRatio = deltaY / containerHeight
      const newScrollY = scrollbarDrag.startScrollY + scrollRatio * maxScrollY
      scrollY.value = Math.max(0, Math.min(maxScrollY, newScrollY))
      updateViewport()
    }
  }
}

function stopScrollbarDrag() {
  scrollbarDrag.isHorizontalDragging = false
  scrollbarDrag.isVerticalDragging = false
  
  // 移除全局事件监听
  document.removeEventListener('mousemove', handleScrollbarDrag)
  document.removeEventListener('mouseup', stopScrollbarDrag)
}

// Clip拖拽事件处理
function handleClipMouseDown(event, clip, track) {
  event.stopPropagation()
  
  clipDrag.isDragging = true
  clipDrag.draggedClip = clip
  clipDrag.draggedTrackId = track.id
  clipDrag.startX = event.data.global.x
  clipDrag.startTime = clip.startTime
  clipDrag.offsetX = 0
  
  // 添加全局事件监听
  document.addEventListener('pointermove', handleClipDrag)
  document.addEventListener('pointerup', stopClipDrag)
  
  console.log('开始拖拽片段:', clip.name)
}

function handleClipDrag(event) {
  if (!clipDrag.isDragging) return
  
  const deltaX = event.clientX - clipDrag.startX
  clipDrag.offsetX = deltaX
  
  // 计算新的时间位置
  const newTime = clipDrag.startTime + (deltaX / (pixelsPerSecond * zoomX.value))
  const snapTime = Math.max(0, Math.round(newTime * 4) / 4) // 1/4秒对齐
  
  // 只有在位置真正改变时才更新
  if (clipDrag.draggedClip && Math.abs(clipDrag.draggedClip.startTime - snapTime) > 0.01) {
    clipDrag.draggedClip.startTime = snapTime
    
    // 节流重新渲染
    clearTimeout(handleClipDrag.timeoutId)
    handleClipDrag.timeoutId = setTimeout(() => {
      createTracks()
      needsRedraw = true
    }, 16) // 约60fps
  }
}

function stopClipDrag() {
  if (clipDrag.isDragging) {
    console.log('停止拖拽片段:', clipDrag.draggedClip?.name, '新位置:', clipDrag.draggedClip?.startTime)
  }
  
  clipDrag.isDragging = false
  clipDrag.draggedClip = null
  clipDrag.draggedTrackId = null
  
  // 移除全局事件监听
  document.removeEventListener('pointermove', handleClipDrag)
  document.removeEventListener('pointerup', stopClipDrag)
}

// 轨道拖拽事件处理
function handleTrackMouseDown(event, track, index) {
  // 只在拖拽手柄区域响应
  if (!event.target.closest('.drag-handle')) return
  
  event.preventDefault()
  event.stopPropagation()
  
  trackDrag.isDragging = true
  trackDrag.draggedTrackId = track.id
  trackDrag.startY = event.clientY
  trackDrag.startOrder = track.order
  
  // 添加全局事件监听
  document.addEventListener('mousemove', handleTrackDrag)
  document.addEventListener('mouseup', stopTrackDrag)
  
  console.log('开始拖拽轨道:', track.name)
}

function handleTrackDrag(event) {
  if (!trackDrag.isDragging) return
  
  const deltaY = event.clientY - trackDrag.startY
  const trackHeightScaled = trackHeight * zoomY.value
  const orderDelta = Math.round(deltaY / trackHeightScaled)
  const newOrder = Math.max(0, Math.min(tracks.value.length - 1, trackDrag.startOrder + orderDelta))
  
  // 找到被拖拽的轨道
  const draggedTrack = tracks.value.find(t => t.id === trackDrag.draggedTrackId)
  if (!draggedTrack) return
  
  // 只在order真正改变时更新
  if (draggedTrack.order !== newOrder) {
    const oldOrder = draggedTrack.order
    
    // 更新其他轨道的order
    tracks.value.forEach(track => {
      if (track.id === trackDrag.draggedTrackId) {
        track.order = newOrder
      } else if (track.order >= Math.min(oldOrder, newOrder) && 
                 track.order <= Math.max(oldOrder, newOrder)) {
        if (newOrder > oldOrder) {
          track.order -= 1
        } else {
          track.order += 1
        }
      }
    })
    
    // 节流重新渲染
    clearTimeout(handleTrackDrag.timeoutId)
    handleTrackDrag.timeoutId = setTimeout(() => {
      createTracks()
      needsRedraw = true
    }, 16) // 约60fps
  }
}

function stopTrackDrag() {
  if (trackDrag.isDragging) {
    const draggedTrack = tracks.value.find(t => t.id === trackDrag.draggedTrackId)
    console.log('停止拖拽轨道:', draggedTrack?.name, '新顺序:', draggedTrack?.order)
  }
  
  trackDrag.isDragging = false
  trackDrag.draggedTrackId = null
  
  // 移除全局事件监听
  document.removeEventListener('mousemove', handleTrackDrag)
  document.removeEventListener('mouseup', stopTrackDrag)
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

// 工具函数
function formatTime(seconds) {
  const mins = Math.floor(seconds / 60)
  const secs = Math.floor(seconds % 60)
  return `${mins}:${secs.toString().padStart(2, '0')}`
}

// 窗口大小调整
function handleResize() {
  if (app && app.renderer && pixiContainer.value) {
    try {
      const newWidth = pixiContainer.value.clientWidth || 800
      const newHeight = pixiContainer.value.clientHeight || 600
      
      // 安全地调整渲染器大小
      app.renderer.resize(newWidth, newHeight)
      
      // 更新视口边界
      updateViewportBounds()
      
      // 重新渲染内容
      createTimeline()
      createTracks()
      updatePlayhead()
      
      console.log(`📐 窗口大小调整: ${newWidth}x${newHeight}`)
    } catch (error) {
      console.warn('⚠️ 窗口大小调整失败:', error)
    }
  }
}

// 键盘事件处理
function handleKeyDown(event) {
  const scrollSpeed = 50
  const containerWidth = pixiContainer.value?.clientWidth || 800
  const containerHeight = (pixiContainer.value?.clientHeight || 600) - timelineHeight
  const maxScrollX = Math.max(0, maxDuration * pixelsPerSecond * zoomX.value - containerWidth)
  const maxScrollY = Math.max(0, tracks.value.length * trackHeight * zoomY.value - containerHeight)
  
  switch(event.key) {
    case 'ArrowLeft':
      event.preventDefault()
      scrollX.value = Math.max(0, scrollX.value - scrollSpeed)
      updateViewport()
      break
    case 'ArrowRight':
      event.preventDefault()
      scrollX.value = Math.min(maxScrollX, scrollX.value + scrollSpeed)
      updateViewport()
      break
    case 'ArrowUp':
      event.preventDefault()
      scrollY.value = Math.max(0, scrollY.value - scrollSpeed)
      updateViewport()
      break
    case 'ArrowDown':
      event.preventDefault()
      scrollY.value = Math.min(maxScrollY, scrollY.value + scrollSpeed)
      updateViewport()
      break
    case 'Home':
      event.preventDefault()
      scrollX.value = 0
      scrollY.value = 0
      updateViewport()
      break
    case 'End':
      event.preventDefault()
      scrollX.value = maxScrollX
      updateViewport()
      break
    case ' ':
      event.preventDefault()
      togglePlayback()
      break
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
  if (app) {
    window.removeEventListener('resize', handleResize)
    window.removeEventListener('keydown', handleKeyDown)
    app.destroy(true)
  }
  
  // 清理滚动条事件监听
  document.removeEventListener('mousemove', handleScrollbarDrag)
  document.removeEventListener('mouseup', stopScrollbarDrag)
})
</script>

<style scoped>
.pixi-track-editor {
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

.pixi-container {
  position: relative;
  width: 100%;
  height: calc(100% - 80px);
  overflow: hidden;
  cursor: grab;
  outline: none; /* 移除焦点时的边框 */
}

.pixi-container:active {
  cursor: grabbing;
}

.pixi-container:focus {
  box-shadow: inset 0 0 0 2px #10b981; /* 获得焦点时的提示 */
}

.scrollbars {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
}

.horizontal-scrollbar,
.vertical-scrollbar {
  position: absolute;
  background: rgba(255, 255, 255, 0.1);
  pointer-events: auto;
}

.horizontal-scrollbar {
  bottom: 0;
  left: 0;
  right: 15px;
  height: 15px;
}

.vertical-scrollbar {
  top: 0;
  right: 0;
  bottom: 15px;
  width: 15px;
}

.scroll-thumb {
  background: rgba(255, 255, 255, 0.3);
  border-radius: 7px;
  transition: background 0.2s;
  cursor: pointer;
  height: 100%;
  width: 100%;
}

.scroll-thumb:hover {
  background: rgba(255, 255, 255, 0.5);
}

.track-sidebar {
  position: absolute;
  left: 0;
  top: 80px;
  width: 200px;
  height: calc(100% - 80px);
  background: rgba(0, 0, 0, 0.8);
  backdrop-filter: blur(10px);
  border-right: 2px solid #333;
  overflow: hidden;
  pointer-events: auto;
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

.track-info.track-dragging {
  background: rgba(16, 185, 129, 0.1);
  border-left: 3px solid #10b981;
  z-index: 1000;
}

.drag-handle {
  width: 20px;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: grab;
  padding: 5px;
  border-radius: 4px;
  transition: background-color 0.2s;
}

.drag-handle:hover {
  background: rgba(255, 255, 255, 0.1);
}

.drag-handle:active {
  cursor: grabbing;
}

.drag-dots {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 2px;
}

.dot {
  width: 3px;
  height: 3px;
  background: #666;
  border-radius: 50%;
}

.drag-handle:hover .dot {
  background: #999;
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
</style>