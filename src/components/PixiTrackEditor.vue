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

// 性能优化相关
let viewportBounds = { left: 0, right: 0, top: 0, bottom: 0 }
let renderCache = new Map()
let lastViewport = { x: 0, y: 0, zoomX: 1, zoomY: 1 }
let needsRedraw = true

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
    if (!track.waveformData.length) {
      track.waveformData = generateWaveformData(track)
    }
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

// 优化的轨道创建 - 只渲染可视区域
function createTracks() {
  tracksContainer.removeChildren()
  updateViewportBounds()
  
  // 只渲染可视范围内的轨道
  for (let index = viewportBounds.startTrack; index <= viewportBounds.endTrack; index++) {
    if (index >= tracks.value.length) break
    
    const track = tracks.value[index]
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
    trackContainer.addChild(trackBg)
    
    // 轨道分割线 - 只渲染可视宽度
    const separator = new PIXI.Graphics()
    separator.lineStyle(1, 0x333333)
    separator.moveTo(viewportBounds.left, trackHeight * zoomY.value)
    separator.lineTo(viewportBounds.right, trackHeight * zoomY.value)
    trackContainer.addChild(separator)
    
    // 创建波形 - 使用缓存和可视区域优化
    if (track.waveformData.length > 0) {
      const waveform = createOptimizedWaveform(track, index)
      if (waveform) {
        trackContainer.addChild(waveform)
      }
    }
    
    tracksContainer.addChild(trackContainer)
  }
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