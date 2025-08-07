<template>
  <div class="timeline-controls">
    <!-- 播放控制器 -->
    <div class="playback-controls">
      <t-button variant="text" size="large" @click="togglePlay">
        <template #icon>
          <component :is="isPlaying ? 'pause' : 'play'" />
        </template>
      </t-button>
      <div class="time-display">{{ currentTime }}</div>
    </div>

    <!-- 时间轴 -->
    <div class="timeline">
      <div class="timeline-ruler">
        <div 
          v-for="mark in timeMarks" 
          :key="mark.time"
          class="time-mark"
          :style="{ left: mark.position + '%' }"
        >
          <div class="mark-line"></div>
          <div class="mark-label">{{ mark.time }}</div>
        </div>
      </div>
      
      <!-- 播放头 -->
      <div 
        class="playhead" 
        :style="{ left: playheadPosition + '%' }"
      ></div>
      
      <!-- 进度条 -->
      <div class="timeline-track" @click="seekToPosition">
        <div 
          class="timeline-progress" 
          :style="{ width: playheadPosition + '%' }"
        ></div>
      </div>
    </div>

    <!-- 右侧图标 -->
    <div class="timeline-icon">
      <div class="icon-placeholder"></div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const isPlaying = ref(false)
const currentTimeSeconds = ref(0)
const totalDuration = ref(43) // 43秒总时长

// 时间格式化
const currentTime = computed(() => {
  const minutes = Math.floor(currentTimeSeconds.value / 60)
  const seconds = Math.floor(currentTimeSeconds.value % 60)
  return `${minutes}:${seconds.toString().padStart(2, '0')}`
})

// 播放头位置百分比
const playheadPosition = computed(() => {
  return (currentTimeSeconds.value / totalDuration.value) * 100
})

// 时间刻度标记
const timeMarks = computed(() => {
  const marks = []
  const interval = 5 // 每5秒一个标记
  
  for (let i = 0; i <= totalDuration.value; i += interval) {
    const minutes = Math.floor(i / 60)
    const seconds = i % 60
    marks.push({
      time: `${minutes}:${seconds.toString().padStart(2, '0')}`,
      position: (i / totalDuration.value) * 100
    })
  }
  
  return marks
})

let playInterval = null

const togglePlay = () => {
  isPlaying.value = !isPlaying.value
  
  if (isPlaying.value) {
    playInterval = setInterval(() => {
      currentTimeSeconds.value += 0.1
      if (currentTimeSeconds.value >= totalDuration.value) {
        currentTimeSeconds.value = totalDuration.value
        isPlaying.value = false
        clearInterval(playInterval)
      }
    }, 100)
  } else {
    clearInterval(playInterval)
  }
}

const seekToPosition = (event) => {
  const rect = event.currentTarget.getBoundingClientRect()
  const clickX = event.clientX - rect.left
  const percentage = clickX / rect.width
  currentTimeSeconds.value = percentage * totalDuration.value
}

onUnmounted(() => {
  if (playInterval) {
    clearInterval(playInterval)
  }
})
</script>

<style scoped>
.timeline-controls {
  display: flex;
  align-items: center;
  padding: 16px 24px;
  background: white;
  gap: 16px;
  border-bottom: 1px solid #e7e7e7;
}

.playback-controls {
  display: flex;
  align-items: center;
  gap: 12px;
  min-width: 120px;
}

.time-display {
  font-family: 'SF Mono', 'Monaco', 'Cascadia Code', monospace;
  font-size: 14px;
  color: #6b7280;
  min-width: 50px;
}

.timeline {
  flex: 1;
  position: relative;
  height: 40px;
}

.timeline-ruler {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 20px;
}

.time-mark {
  position: absolute;
  top: 0;
}

.mark-line {
  width: 1px;
  height: 8px;
  background: #d1d5db;
}

.mark-label {
  position: absolute;
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
  font-size: 11px;
  color: #9ca3af;
  white-space: nowrap;
}

.timeline-track {
  position: absolute;
  bottom: 8px;
  left: 0;
  right: 0;
  height: 4px;
  background: #e5e7eb;
  border-radius: 2px;
  cursor: pointer;
}

.timeline-progress {
  height: 100%;
  background: #4f46e5;
  border-radius: 2px;
  transition: width 0.1s ease;
}

.playhead {
  position: absolute;
  bottom: 4px;
  width: 2px;
  height: 12px;
  background: #4f46e5;
  border-radius: 1px;
  transform: translateX(-50%);
  pointer-events: none;
  z-index: 2;
}

.timeline-icon {
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.icon-placeholder {
  width: 32px;
  height: 32px;
  background: linear-gradient(135deg, #60a5fa, #3b82f6);
  border-radius: 8px;
  opacity: 0.8;
}
</style>