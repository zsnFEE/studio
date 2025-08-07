<template>
  <div 
    class="waveform-track" 
    :class="{ selected: isSelected }"
    @click="$emit('select', track.id)"
  >
    <!-- 轨道控制区域 -->
    <div class="track-controls">
      <div class="track-icon" :style="{ backgroundColor: track.color }">
        <div class="track-type">{{ track.type }}</div>
      </div>
      
      <div class="track-info">
        <div class="track-name">{{ track.name }}</div>
      </div>
      
      <div class="control-buttons">
        <t-button 
          size="small" 
          variant="text"
          :theme="track.isSolo ? 'primary' : 'default'"
          @click.stop="$emit('toggle-solo', track.id)"
        >
          S
        </t-button>
        <t-button 
          size="small" 
          variant="text"
          :theme="track.isMuted ? 'danger' : 'default'"
          @click.stop="$emit('toggle-mute', track.id)"
        >
          🔇
        </t-button>
      </div>
    </div>

    <!-- 波形显示区域 -->
    <div class="waveform-area">
      <svg 
        class="waveform-svg" 
        :viewBox="`0 0 ${waveformWidth} ${waveformHeight}`"
        preserveAspectRatio="none"
      >
        <g v-for="segment in track.segments" :key="`${segment.start}-${segment.duration}`">
          <path
            :d="generateWaveformPath(segment)"
            :fill="track.color"
            :opacity="track.isMuted ? 0.3 : 0.8"
          />
        </g>
      </svg>
    </div>
  </div>
</template>

<script setup>
import { computed } from 'vue'

const props = defineProps({
  track: {
    type: Object,
    required: true
  },
  isSelected: {
    type: Boolean,
    default: false
  }
})

defineEmits(['select', 'toggle-solo', 'toggle-mute'])

const waveformWidth = 1000
const waveformHeight = 60
const totalDuration = 120 // 总时长，用于计算位置

// 生成波形路径
const generateWaveformPath = (segment) => {
  const startX = (segment.start / totalDuration) * waveformWidth
  const width = (segment.duration / totalDuration) * waveformWidth
  const centerY = waveformHeight / 2
  const maxAmplitude = (waveformHeight / 2) * segment.amplitude
  
  // 生成随机波形数据
  const points = 100
  const stepX = width / points
  let path = `M ${startX} ${centerY}`
  
  for (let i = 0; i <= points; i++) {
    const x = startX + (i * stepX)
    // 生成伪随机波形
    const frequency1 = Math.sin((i / points) * Math.PI * 8) * 0.6
    const frequency2 = Math.sin((i / points) * Math.PI * 20) * 0.3
    const frequency3 = Math.sin((i / points) * Math.PI * 50) * 0.1
    const amplitude = (frequency1 + frequency2 + frequency3) * maxAmplitude
    
    const y1 = centerY - Math.abs(amplitude)
    const y2 = centerY + Math.abs(amplitude)
    
    if (i === 0) {
      path += ` L ${x} ${y1}`
    } else {
      path += ` L ${x} ${y1}`
    }
  }
  
  // 绘制下半部分
  for (let i = points; i >= 0; i--) {
    const x = startX + (i * stepX)
    const frequency1 = Math.sin((i / points) * Math.PI * 8) * 0.6
    const frequency2 = Math.sin((i / points) * Math.PI * 20) * 0.3
    const frequency3 = Math.sin((i / points) * Math.PI * 50) * 0.1
    const amplitude = (frequency1 + frequency2 + frequency3) * maxAmplitude
    
    const y2 = centerY + Math.abs(amplitude)
    path += ` L ${x} ${y2}`
  }
  
  path += ' Z'
  return path
}
</script>

<style scoped>
.waveform-track {
  display: flex;
  min-height: 80px;
  border-bottom: 1px solid #f3f4f6;
  cursor: pointer;
  transition: background-color 0.2s;
}

.waveform-track:hover {
  background-color: #f9fafb;
}

.waveform-track.selected {
  background-color: #eff6ff;
  border-left: 3px solid #4f46e5;
}

.track-controls {
  display: flex;
  align-items: center;
  padding: 16px;
  gap: 12px;
  min-width: 200px;
  background: #fafbfc;
  border-right: 1px solid #e5e7eb;
}

.track-icon {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-weight: 600;
  font-size: 14px;
}

.track-info {
  flex: 1;
}

.track-name {
  font-size: 14px;
  font-weight: 500;
  color: #374151;
}

.control-buttons {
  display: flex;
  gap: 4px;
}

.waveform-area {
  flex: 1;
  position: relative;
  overflow: hidden;
}

.waveform-svg {
  width: 100%;
  height: 100%;
}
</style>