<template>
  <div class="simple-2d-waveform">
    <!-- 简化的控制面板 -->
    <div class="controls-panel">
      <div class="panel-left">
        <h3>🎵 音频轨道</h3>
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
            />
          </div>
        </div>
      </div>
      
      <div class="panel-right">
        <div class="track-info">
          <span>轨道数: {{ tracks.length }}</span>
        </div>
      </div>
    </div>

    <!-- 2D 波形显示区域 -->
    <div class="waveform-container">
      <div 
        v-for="(track, index) in tracks" 
        :key="track.id"
        class="track-row"
        :class="{ 'track-selected': selectedTrack === track.id }"
        @click="selectTrack(track.id)"
      >
        <!-- 轨道信息面板 -->
        <div class="track-info-panel">
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
              {{ track.isSolo ? 'S' : 'S' }}
            </t-button>
            <t-button 
              size="small" 
              :theme="track.isMuted ? 'danger' : 'default'"
              @click="toggleMute(track.id)"
            >
              {{ track.isMuted ? 'M' : 'M' }}
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

        <!-- 2D 波形显示 -->
        <div class="waveform-display">
          <canvas 
            :ref="el => canvasRefs[track.id] = el"
            class="waveform-canvas"
            :width="800"
            :height="120"
          ></canvas>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick, reactive } from 'vue'

// 响应式状态
const selectedTrack = ref(1)
const globalPlaying = ref(false)
const masterVolume = ref(75)
const canvasRefs = reactive({})

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
    isMuted: false,
    isPlaying: false
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
    isMuted: false,
    isPlaying: false
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
    waveformData: [],
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
    waveformData: [],
    isSolo: false,
    isMuted: false,
    isPlaying: false
  }
])

// 生成简化的波形数据
function generateSimpleWaveform(track) {
  const pointsPerSecond = 50 // 降低密度
  const totalPoints = track.duration * pointsPerSecond
  const waveform = []
  
  for (let i = 0; i < totalPoints; i++) {
    const time = i / pointsPerSecond
    let amplitude = 0
    
    // 根据轨道类型生成不同的波形特征
    switch(track.type) {
      case 'LEAD':
        amplitude = Math.sin(time * 4) * 0.8
        amplitude += Math.sin(time * 8) * 0.3
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
        amplitude = Math.sin(time * 3) * 0.7
        break
      case 'PAD':
        amplitude = Math.sin(time * 2) * 0.4
        amplitude += Math.sin(time * 2.5) * 0.3
        break
      default:
        amplitude = Math.sin(time * 4) * 0.6
    }
    
    amplitude = Math.max(0, Math.min(1, Math.abs(amplitude)))
    waveform.push(amplitude)
  }
  
  return waveform
}

// 绘制2D波形
function draw2DWaveform(track) {
  const canvas = canvasRefs[track.id]
  if (!canvas || !track.waveformData.length) return
  
  const ctx = canvas.getContext('2d')
  const width = canvas.width
  const height = canvas.height
  
  // 清空画布
  ctx.fillStyle = '#1a1a1a'
  ctx.fillRect(0, 0, width, height)
  
  // 绘制网格线
  ctx.strokeStyle = '#333'
  ctx.lineWidth = 1
  ctx.setLineDash([2, 2])
  
  // 水平网格线
  for (let i = 0; i <= 4; i++) {
    const y = (height / 4) * i
    ctx.beginPath()
    ctx.moveTo(0, y)
    ctx.lineTo(width, y)
    ctx.stroke()
  }
  
  // 中心线
  ctx.strokeStyle = '#555'
  ctx.setLineDash([])
  ctx.beginPath()
  ctx.moveTo(0, height / 2)
  ctx.lineTo(width, height / 2)
  ctx.stroke()
  
  // 绘制波形
  const waveformData = track.waveformData
  const pointWidth = width / waveformData.length
  
  // 设置波形样式
  ctx.fillStyle = track.color
  ctx.globalAlpha = track.isMuted ? 0.3 : 0.8
  
  // 绘制波形条
  for (let i = 0; i < waveformData.length; i++) {
    const amplitude = waveformData[i]
    const x = i * pointWidth
    const barHeight = amplitude * (height / 2)
    
    // 上半部分（正波形）
    ctx.fillRect(x, height / 2 - barHeight, pointWidth - 1, barHeight)
    // 下半部分（负波形镜像）
    ctx.fillRect(x, height / 2, pointWidth - 1, barHeight)
  }
  
  // 如果正在播放，添加播放指示器
  if (globalPlaying.value && !track.isMuted) {
    const playbackPosition = (Date.now() / 100) % width
    ctx.globalAlpha = 0.8
    ctx.fillStyle = '#fff'
    ctx.fillRect(playbackPosition, 0, 2, height)
  }
  
  ctx.globalAlpha = 1
}

// 初始化所有轨道的波形数据
function initializeWaveforms() {
  tracks.value.forEach(track => {
    if (!track.waveformData.length) {
      track.waveformData = generateSimpleWaveform(track)
    }
  })
}

// 渲染所有波形
function renderAllWaveforms() {
  tracks.value.forEach(track => {
    draw2DWaveform(track)
  })
}

// 动画循环
let animationId = null
function animate() {
  renderAllWaveforms()
  animationId = requestAnimationFrame(animate)
}

// 轨道控制函数
function selectTrack(trackId) {
  selectedTrack.value = trackId
}

function toggleSolo(trackId) {
  const track = tracks.value.find(t => t.id === trackId)
  if (track) {
    track.isSolo = !track.isSolo
    // 如果有轨道Solo，其他轨道静音
    if (track.isSolo) {
      tracks.value.forEach(t => {
        if (t.id !== trackId) t.isMuted = true
      })
    } else {
      tracks.value.forEach(t => t.isMuted = false)
    }
  }
}

function toggleMute(trackId) {
  const track = tracks.value.find(t => t.id === trackId)
  if (track) {
    track.isMuted = !track.isMuted
  }
}

function toggleGlobalPlayback() {
  globalPlaying.value = !globalPlaying.value
}

// 生命周期
onMounted(async () => {
  await nextTick()
  initializeWaveforms()
  renderAllWaveforms()
  animate()
})

onUnmounted(() => {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
})
</script>

<style scoped>
.simple-2d-waveform {
  width: 100%;
  background: linear-gradient(135deg, #0a0a0a 0%, #1a1a2e 100%);
  border-radius: 16px;
  overflow: hidden;
  border: 2px solid #333;
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
  margin: 0;
  color: #fff;
  font-size: 18px;
}

.master-controls {
  display: flex;
  align-items: center;
  gap: 20px;
}

.master-volume {
  display: flex;
  align-items: center;
  gap: 10px;
  color: #fff;
}

.master-volume .t-slider {
  width: 120px;
}

.track-info {
  color: #999;
  font-size: 14px;
}

.waveform-container {
  padding: 0;
}

.track-row {
  display: flex;
  border-bottom: 1px solid #333;
  transition: background-color 0.2s;
  cursor: pointer;
}

.track-row:hover {
  background: rgba(255, 255, 255, 0.02);
}

.track-row.track-selected {
  background: rgba(16, 185, 129, 0.1);
  border-left: 4px solid #10b981;
}

.track-info-panel {
  width: 200px;
  padding: 15px;
  background: rgba(0, 0, 0, 0.3);
  display: flex;
  flex-direction: column;
  gap: 10px;
  border-right: 1px solid #333;
}

.track-header h4 {
  margin: 0;
  font-size: 16px;
}

.track-type {
  font-size: 12px;
  background: rgba(255, 255, 255, 0.1);
  padding: 2px 8px;
  border-radius: 12px;
  color: #ccc;
}

.track-controls {
  display: flex;
  gap: 8px;
}

.track-controls .t-button {
  width: 32px;
  height: 32px;
  font-weight: bold;
}

.volume-control {
  height: 60px;
  display: flex;
  justify-content: center;
}

.waveform-display {
  flex: 1;
  padding: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.waveform-canvas {
  border: 1px solid #444;
  border-radius: 4px;
  background: #0f0f0f;
}
</style>