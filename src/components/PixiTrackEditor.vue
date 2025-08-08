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
        <div 
          v-for="(track, index) in tracks" 
          :key="track.id"
          class="track-info"
          :style="{ 
            top: (index * 120) + 'px',
            height: '120px'
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

      <!-- 时间轴和轨道区域 -->
      <div class="timeline-and-tracks">
        <!-- 时间轴区域 -->
        <div class="timeline-header">
          <div class="timeline-content">
            <div class="time-marker" v-for="t in 10" :key="t" :style="{ left: (t * 80) + 'px' }">
              {{ t }}s
            </div>
          </div>
        </div>
        
        <!-- 轨道渲染区域 -->
        <div class="tracks-container">
          <div 
            v-for="(track, index) in tracks"
            :key="track.id"
            class="track-row"
            :style="{ 
              top: (index * 120) + 'px',
              height: '120px'
            }"
          >
            <div class="track-background" :style="{ backgroundColor: index % 2 ? '#1e1e1e' : '#252525' }"></div>
            <div 
              v-for="clip in track.clips"
              :key="clip.id"
              class="audio-clip"
              :style="{
                left: (clip.startTime * 10) + 'px',
                width: (clip.duration * 10) + 'px',
                backgroundColor: clip.color
              }"
            >
              <div class="clip-name">{{ clip.name }}</div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

// 响应式状态
const isPlaying = ref(false)
const currentTime = ref(0)
const zoomX = ref(1)
const zoomY = ref(1)

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
      { id: 2, name: '主旋律_2', startTime: 20, duration: 12, color: '#059669' }
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
      { id: 3, name: '鼓点_1', startTime: 0, duration: 30, color: '#f59e0b' }
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
      { id: 4, name: '贝斯_1', startTime: 0, duration: 25, color: '#3b82f6' }
    ]
  }
])

// 事件处理
function togglePlayback() {
  isPlaying.value = !isPlaying.value
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

function formatTime(seconds) {
  const mins = Math.floor(seconds / 60)
  const secs = Math.floor(seconds % 60)
  return `${mins}:${secs.toString().padStart(2, '0')}`
}
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

.tracks-container {
  flex: 1;
  position: relative;
  overflow: auto;
  background: #1a1a1a;
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
  height: 80px;
  top: 20px;
  border-radius: 4px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  cursor: pointer;
  display: flex;
  align-items: center;
  padding: 8px;
  box-sizing: border-box;
  transition: opacity 0.2s;
}

.audio-clip:hover {
  opacity: 0.8;
}

.clip-name {
  color: white;
  font-size: 12px;
  font-weight: bold;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
}
</style>