<template>
  <div class="simple-track-editor">
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
          />
          <span>垂直缩放:</span>
          <t-slider 
            v-model="zoomY" 
            :min="0.5" 
            :max="3" 
            :step="0.1"
          />
        </div>
      </div>
      
      <div class="panel-right">
        <div class="view-info">
          <span>时间: {{ formatTime(0) }} - {{ formatTime(60) }}</span>
          <span>轨道: {{ tracks.length }}</span>
        </div>
      </div>
    </div>

    <!-- 主工作区域 -->
    <div class="main-workspace">
      <!-- 轨道信息侧边栏 -->
      <div class="track-sidebar">
        <div class="timeline-placeholder">时间轴区域</div>
        <div class="tracks-sidebar-content" :style="{ transform: `translateY(-${scrollY}px)` }">
          <div 
            v-for="(track, index) in tracks" 
            :key="track.id"
            class="track-info"
            :style="{ 
              top: (index * trackHeight * zoomY) + 'px',
              height: (trackHeight * zoomY) + 'px'
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

      <!-- 时间轴和轨道区域 -->
      <div class="timeline-and-tracks">
        <!-- 时间轴区域 -->
        <div class="timeline-header">
          <div class="timeline-content" :style="{ transform: `translateX(-${scrollX}px)` }">
            <div 
              v-for="t in 30" 
              :key="t" 
              class="time-marker" 
              :style="{ left: (t * pixelsPerSecond * zoomX) + 'px' }"
            >
              {{ t }}s
            </div>
            <div class="playhead" :style="{ left: (currentTime * pixelsPerSecond * zoomX) + 'px' }"></div>
          </div>
        </div>
        
        <!-- 轨道渲染区域 -->
        <div class="tracks-container" @wheel="handleWheel" @scroll="handleScroll">
          <div 
            class="tracks-content" 
            :style="{ 
              transform: `translate(-${scrollX}px, -${scrollY}px)`,
              width: (30 * pixelsPerSecond * zoomX) + 'px',
              height: (tracks.length * trackHeight * zoomY) + 'px'
            }"
          >
            <div 
              v-for="(track, index) in tracks"
              :key="track.id"
              class="track-row"
              :style="{ 
                top: (index * trackHeight * zoomY) + 'px',
                height: (trackHeight * zoomY) + 'px'
              }"
            >
              <div class="track-background" :style="{ backgroundColor: index % 2 ? '#1e1e1e' : '#252525' }"></div>
              <div 
                v-for="clip in track.clips"
                :key="clip.id"
                class="audio-clip"
                :style="{
                  left: (clip.startTime * pixelsPerSecond * zoomX) + 'px',
                  width: (clip.duration * pixelsPerSecond * zoomX) + 'px',
                  backgroundColor: clip.color,
                  height: (trackHeight * zoomY - 40) + 'px'
                }"
                @click="selectClip(clip)"
              >
                <div class="clip-name">{{ clip.name }}</div>
                <div class="clip-duration">{{ clip.duration }}s</div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'

// 响应式状态
const isPlaying = ref(false)
const currentTime = ref(0)
const zoomX = ref(1)
const zoomY = ref(1)
const selectedClip = ref(null)
const scrollX = ref(0)
const scrollY = ref(0)

// 布局常量
const trackHeight = 120
const timelineHeight = 60
const pixelsPerSecond = 20 // 时间轴像素比例

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
      { id: 1, name: '主旋律_1', startTime: 0, duration: 15, color: '#10b981' },
      { id: 2, name: '主旋律_2', startTime: 20, duration: 12, color: '#059669' },
      { id: 3, name: '主旋律_3', startTime: 40, duration: 18, color: '#047857' }
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
      { id: 4, name: '鼓点_1', startTime: 0, duration: 8, color: '#f59e0b' },
      { id: 5, name: '鼓点_2', startTime: 12, duration: 10, color: '#d97706' },
      { id: 6, name: '鼓点_3', startTime: 25, duration: 8, color: '#f59e0b' },
      { id: 7, name: '鼓点_4', startTime: 40, duration: 12, color: '#b45309' }
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
      { id: 8, name: '贝斯_1', startTime: 0, duration: 25, color: '#3b82f6' },
      { id: 9, name: '贝斯_2', startTime: 30, duration: 20, color: '#2563eb' }
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
      { id: 10, name: '和弦_1', startTime: 5, duration: 18, color: '#8b5cf6' },
      { id: 11, name: '和弦_2', startTime: 30, duration: 15, color: '#7c3aed' },
      { id: 12, name: '和弦_3', startTime: 50, duration: 20, color: '#6d28d9' }
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
      { id: 13, name: '合成器_1', startTime: 10, duration: 20, color: '#ef4444' },
      { id: 14, name: '合成器_2', startTime: 35, duration: 15, color: '#dc2626' }
    ]
  },
  {
    id: 6,
    name: '氛围音',
    type: 'PAD',
    color: '#06b6d4',
    volume: 45,
    isSolo: false,
    isMuted: false,
    clips: [
      { id: 15, name: '氛围_1', startTime: 0, duration: 30, color: '#06b6d4' },
      { id: 16, name: '氛围_2', startTime: 35, duration: 25, color: '#0891b2' }
    ]
  },
  {
    id: 7,
    name: '人声',
    type: 'VOCAL',
    color: '#f97316',
    volume: 90,
    isSolo: false,
    isMuted: false,
    clips: [
      { id: 17, name: '主歌', startTime: 8, duration: 16, color: '#f97316' },
      { id: 18, name: '副歌', startTime: 30, duration: 20, color: '#ea580c' },
      { id: 19, name: '桥段', startTime: 55, duration: 10, color: '#c2410c' }
    ]
  },
  {
    id: 8,
    name: '打击乐',
    type: 'PERC2',
    color: '#84cc16',
    volume: 80,
    isSolo: false,
    isMuted: false,
    clips: [
      { id: 20, name: '镲片', startTime: 0, duration: 60, color: '#84cc16' },
      { id: 21, name: '手鼓', startTime: 15, duration: 25, color: '#65a30d' }
    ]
  }
])

// 播放模拟
let playInterval = null

// 事件处理
function togglePlayback() {
  isPlaying.value = !isPlaying.value
  
  if (isPlaying.value) {
    playInterval = setInterval(() => {
      currentTime.value += 0.1
      if (currentTime.value >= 60) {
        currentTime.value = 0
      }
    }, 100)
  } else {
    if (playInterval) {
      clearInterval(playInterval)
      playInterval = null
    }
  }
}

function toggleSolo(trackId) {
  const track = tracks.value.find(t => t.id === trackId)
  if (track) {
    track.isSolo = !track.isSolo
    // 当某个轨道开启Solo时，其他轨道自动Mute
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

function selectClip(clip) {
  selectedClip.value = clip
  console.log('选中片段:', clip.name)
}

// 滚动和缩放处理
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
  } else {
    // 滚动操作
    const scrollSpeed = 30
    
    if (event.shiftKey) {
      // Shift+滚轮：水平滚动
      const maxScrollX = Math.max(0, 30 * pixelsPerSecond * zoomX.value - 800)
      scrollX.value = Math.max(0, Math.min(maxScrollX, scrollX.value + event.deltaY))
    } else {
      // 普通滚轮：垂直滚动
      const maxScrollY = Math.max(0, tracks.value.length * trackHeight * zoomY.value - 400)
      scrollY.value = Math.max(0, Math.min(maxScrollY, scrollY.value + event.deltaY))
    }
  }
}

function handleScroll(event) {
  // 可选：处理容器滚动事件
}

function formatTime(seconds) {
  const mins = Math.floor(seconds / 60)
  const secs = Math.floor(seconds % 60)
  return `${mins}:${secs.toString().padStart(2, '0')}`
}

// 组件初始化
onMounted(() => {
  console.log('✅ 简单轨道编辑器加载成功')
})

// 组件卸载时清理定时器
import { onUnmounted } from 'vue'
onUnmounted(() => {
  if (playInterval) {
    clearInterval(playInterval)
  }
})
</script>

<style scoped>
.simple-track-editor {
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

.timeline-and-tracks {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.timeline-header {
  width: 100%;
  height: 60px;
  background: rgba(255, 255, 255, 0.02);
  border-bottom: 1px solid #333;
  position: relative;
  overflow: hidden;
}

.timeline-content {
  position: relative;
  height: 100%;
}

.time-marker {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  color: #fff;
  font-size: 12px;
  font-family: 'Courier New', monospace;
}

.playhead {
  position: absolute;
  top: 0;
  width: 2px;
  height: 100%;
  background: #ff4444;
  z-index: 10;
  transition: left 0.1s ease;
}

.tracks-container {
  flex: 1;
  position: relative;
  overflow: hidden;
  background: #1a1a1a;
}

.tracks-content {
  position: relative;
  min-height: 100%;
  min-width: 100%;
}

.tracks-sidebar-content {
  position: relative;
  overflow: hidden;
}

.track-row {
  position: absolute;
  width: 100%;
  border-bottom: 1px solid #333;
}

.track-background {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
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

.audio-clip {
  position: absolute;
  top: 20px;
  border-radius: 4px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  cursor: pointer;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 8px;
  box-sizing: border-box;
  transition: all 0.2s;
}

.audio-clip:hover {
  opacity: 0.8;
  transform: translateY(-2px);
}

.clip-name {
  color: white;
  font-size: 12px;
  font-weight: bold;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
  margin-bottom: 4px;
}

.clip-duration {
  color: rgba(255, 255, 255, 0.8);
  font-size: 10px;
}
</style>