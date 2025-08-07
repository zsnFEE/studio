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
      @wheel="handleWheel"
      @mousedown="handleMouseDown"
      @mousemove="handleMouseMove"
      @mouseup="handleMouseUp"
      @mouseleave="handleMouseUp"
    >
      <!-- 滚动条 -->
      <div class="scrollbars">
        <div 
          class="horizontal-scrollbar"
          @mousedown="startHorizontalScroll"
        >
          <div 
            class="scroll-thumb"
            :style="horizontalThumbStyle"
          ></div>
        </div>
        <div 
          class="vertical-scrollbar"
          @mousedown="startVerticalScroll"
        >
          <div 
            class="scroll-thumb"
            :style="verticalThumbStyle"
          ></div>
        </div>
      </div>
    </div>

    <!-- 轨道信息侧边栏 -->
    <div class="track-sidebar">
      <div 
        v-for="(track, index) in tracks" 
        :key="track.id"
        class="track-info"
        :style="{ top: (index * trackHeight * zoomY) + 'px' }"
      >
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

// PixiJS 相关变量
let app = null
let mainContainer = null
let tracksContainer = null
let timelineContainer = null
let playheadLine = null

// 轨道数据
const tracks = ref([
  {
    id: 1,
    name: '华丽主旋律',
    type: 'LEAD',
    color: '#10b981',
    duration: 45,
    volume: 85,
    waveformData: [],
    isSolo: false,
    isMuted: false
  },
  {
    id: 2,
    name: '节奏鼓点',
    type: 'PERC',
    color: '#f59e0b',
    duration: 30,
    volume: 95,
    waveformData: [],
    isSolo: false,
    isMuted: false
  },
  {
    id: 3,
    name: '管乐和弦',
    type: 'WIND',
    color: '#8b5cf6',
    duration: 35,
    volume: 90,
    waveformData: [],
    isSolo: false,
    isMuted: false
  },
  {
    id: 4,
    name: '深邃贝斯',
    type: 'BASS',
    color: '#3b82f6',
    duration: 40,
    volume: 70,
    waveformData: [],
    isSolo: false,
    isMuted: false
  },
  {
    id: 5,
    name: '天籁和声',
    type: 'PAD',
    color: '#ef4444',
    duration: 50,
    volume: 55,
    waveformData: [],
    isSolo: false,
    isMuted: false
  }
])

// 计算属性
const viewportStartTime = computed(() => scrollX.value / (pixelsPerSecond * zoomX.value))
const viewportEndTime = computed(() => {
  const containerWidth = pixiContainer.value?.clientWidth || 800
  return viewportStartTime.value + (containerWidth / (pixelsPerSecond * zoomX.value))
})

const horizontalThumbStyle = computed(() => {
  const containerWidth = pixiContainer.value?.clientWidth || 800
  const contentWidth = maxDuration * pixelsPerSecond * zoomX.value
  const thumbWidth = Math.max(20, (containerWidth / contentWidth) * containerWidth)
  const thumbLeft = (scrollX.value / contentWidth) * containerWidth
  
  return {
    width: thumbWidth + 'px',
    left: thumbLeft + 'px'
  }
})

const verticalThumbStyle = computed(() => {
  const containerHeight = (pixiContainer.value?.clientHeight || 600) - timelineHeight
  const contentHeight = tracks.value.length * trackHeight * zoomY.value
  const thumbHeight = Math.max(20, (containerHeight / contentHeight) * containerHeight)
  const thumbTop = (scrollY.value / contentHeight) * containerHeight
  
  return {
    height: thumbHeight + 'px',
    top: thumbTop + 'px'
  }
})

// 生成波形数据
function generateWaveformData(track) {
  const pointsPerSecond = 50
  const totalPoints = track.duration * pointsPerSecond
  const waveform = []
  
  for (let i = 0; i < totalPoints; i++) {
    const time = i / pointsPerSecond
    let amplitude = 0
    
    switch(track.type) {
      case 'LEAD':
        amplitude = Math.sin(time * 4) * 0.8 + Math.sin(time * 8) * 0.3
        break
      case 'BASS':
        amplitude = Math.sign(Math.sin(time * 2)) * 0.9
        break
      case 'PERC':
        const beat = Math.floor(time * 4) % 4
        if (beat === 0) {
          amplitude = Math.exp(-((time % 1) * 8)) * 1.2
        } else if (beat === 2) {
          amplitude = Math.exp(-((time % 1) * 6)) * 0.8
        }
        break
      case 'WIND':
        amplitude = Math.sin(time * 3 + Math.sin(time * 0.5)) * 0.7
        break
      case 'PAD':
        amplitude = Math.sin(time * 2) * 0.4 + Math.sin(time * 2.5) * 0.3
        break
      default:
        amplitude = Math.sin(time * 4) * 0.6
    }
    
    amplitude = Math.max(0, Math.min(1, Math.abs(amplitude)))
    waveform.push(amplitude)
  }
  
  return waveform
}

// 初始化 PixiJS
async function initPixi() {
  const container = pixiContainer.value
  if (!container) return

  app = new PIXI.Application({
    width: container.clientWidth,
    height: container.clientHeight,
    backgroundColor: 0x1a1a1a,
    antialias: true,
    resolution: window.devicePixelRatio || 1,
    autoDensity: true
  })

  container.appendChild(app.view)

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
  playheadLine.lineTo(0, container.clientHeight)
  mainContainer.addChild(playheadLine)

  // 初始化轨道数据
  initializeTracks()
  
  // 创建时间线
  createTimeline()
  
  // 创建轨道
  createTracks()
  
  // 开始渲染循环
  startRenderLoop()
}

// 初始化轨道数据
function initializeTracks() {
  tracks.value.forEach(track => {
    if (!track.waveformData.length) {
      track.waveformData = generateWaveformData(track)
    }
  })
}

// 创建时间线
function createTimeline() {
  timelineContainer.removeChildren()
  
  const timelineBackground = new PIXI.Graphics()
  timelineBackground.beginFill(0x2a2a2a)
  timelineBackground.drawRect(0, 0, maxDuration * pixelsPerSecond * zoomX.value, timelineHeight)
  timelineBackground.endFill()
  timelineContainer.addChild(timelineBackground)
  
  // 时间刻度
  const timeStep = Math.max(1, Math.floor(10 / zoomX.value)) // 动态调整时间间隔
  for (let t = 0; t <= maxDuration; t += timeStep) {
    const x = t * pixelsPerSecond * zoomX.value
    
    // 主要刻度线
    const majorTick = new PIXI.Graphics()
    majorTick.lineStyle(1, 0x666666)
    majorTick.moveTo(x, timelineHeight - 20)
    majorTick.lineTo(x, timelineHeight)
    timelineContainer.addChild(majorTick)
    
    // 时间标签
    const timeText = new PIXI.Text(formatTime(t), {
      fontSize: 12,
      fill: 0xffffff
    })
    timeText.x = x + 2
    timeText.y = timelineHeight - 35
    timelineContainer.addChild(timeText)
    
    // 次要刻度线
    if (zoomX.value > 0.5) {
      for (let subT = 0.2; subT < timeStep && subT < 1; subT += 0.2) {
        const subX = (t + subT) * pixelsPerSecond * zoomX.value
        if (subX < maxDuration * pixelsPerSecond * zoomX.value) {
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

// 创建轨道
function createTracks() {
  tracksContainer.removeChildren()
  
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
    
    // 创建波形
    if (track.waveformData.length > 0) {
      const waveform = createWaveform(track)
      trackContainer.addChild(waveform)
    }
    
    tracksContainer.addChild(trackContainer)
  })
}

// 创建波形图形
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

// 更新缩放
function updateZoom() {
  createTimeline()
  createTracks()
  updatePlayhead()
}

// 更新播放头位置
function updatePlayhead() {
  if (playheadLine) {
    playheadLine.x = currentTime.value * pixelsPerSecond * zoomX.value - scrollX.value
  }
}

// 更新视口
function updateViewport() {
  if (mainContainer) {
    mainContainer.x = -scrollX.value
    mainContainer.y = -scrollY.value
  }
  updatePlayhead()
}

// 渲染循环
function startRenderLoop() {
  function animate() {
    if (isPlaying.value) {
      currentTime.value += 0.016 // 约60fps
      
      // 自动滚动跟随播放头
      const playheadX = currentTime.value * pixelsPerSecond * zoomX.value
      const containerWidth = pixiContainer.value?.clientWidth || 800
      
      if (playheadX - scrollX.value > containerWidth * 0.8) {
        scrollX.value = playheadX - containerWidth * 0.2
      }
      
      updatePlayhead()
    }
    
    requestAnimationFrame(animate)
  }
  animate()
}

// 事件处理
function togglePlayback() {
  isPlaying.value = !isPlaying.value
}

function handleWheel(event) {
  event.preventDefault()
  
  if (event.ctrlKey || event.metaKey) {
    // 缩放
    const zoomFactor = event.deltaY > 0 ? 0.9 : 1.1
    if (event.shiftKey) {
      zoomY.value = Math.max(0.5, Math.min(3, zoomY.value * zoomFactor))
    } else {
      zoomX.value = Math.max(0.1, Math.min(5, zoomX.value * zoomFactor))
    }
    updateZoom()
  } else {
    // 滚动
    if (event.shiftKey) {
      scrollX.value = Math.max(0, scrollX.value + event.deltaY)
    } else {
      scrollY.value = Math.max(0, scrollY.value + event.deltaY)
    }
    updateViewport()
  }
}

function handleMouseDown(event) {
  mouse.isDown = true
  mouse.lastX = event.clientX
  mouse.lastY = event.clientY
  
  // 点击设置播放位置
  const rect = pixiContainer.value.getBoundingClientRect()
  const localX = event.clientX - rect.left + scrollX.value
  const clickTime = localX / (pixelsPerSecond * zoomX.value)
  
  if (event.clientY - rect.top < timelineHeight) {
    currentTime.value = Math.max(0, Math.min(maxDuration, clickTime))
    updatePlayhead()
  }
}

function handleMouseMove(event) {
  if (mouse.isDown) {
    const deltaX = event.clientX - mouse.lastX
    const deltaY = event.clientY - mouse.lastY
    
    scrollX.value = Math.max(0, scrollX.value - deltaX)
    scrollY.value = Math.max(0, scrollY.value - deltaY)
    
    mouse.lastX = event.clientX
    mouse.lastY = event.clientY
    mouse.isDragging = true
    
    updateViewport()
  }
}

function handleMouseUp() {
  mouse.isDown = false
  mouse.isDragging = false
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
  if (app && pixiContainer.value) {
    app.renderer.resize(pixiContainer.value.clientWidth, pixiContainer.value.clientHeight)
    createTimeline()
    createTracks()
  }
}

// 生命周期
onMounted(async () => {
  await nextTick()
  await initPixi()
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  if (app) {
    window.removeEventListener('resize', handleResize)
    app.destroy(true)
  }
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
}

.pixi-container:active {
  cursor: grabbing;
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
  height: 120px;
  padding: 15px;
  box-sizing: border-box;
  border-bottom: 1px solid #333;
  display: flex;
  flex-direction: column;
  gap: 10px;
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