<template>
  <div 
    class="audio-clip-item" 
    :class="{ selected: isSelected }"
    @click="$emit('select', clip.id)"
  >
    <div class="clip-thumbnail">
      <img :src="clipThumbnail" :alt="clip.title" />
      <div class="duration-overlay">{{ clip.duration }}</div>
      <div class="total-duration">{{ clip.totalDuration }}</div>
    </div>
    
    <div class="clip-info">
      <h4 class="clip-title">{{ clip.title }}</h4>
      <div class="clip-tags">
        <t-tag 
          v-for="(tag, index) in clip.tags" 
          :key="index"
          size="small"
          theme="success"
          variant="light"
        >
          {{ tag }}
        </t-tag>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed } from 'vue'

const props = defineProps({
  clip: {
    type: Object,
    required: true
  },
  isSelected: {
    type: Boolean,
    default: false
  }
})

defineEmits(['select'])

// 生成一个简单的缩略图占位符
const clipThumbnail = computed(() => {
  return `data:image/svg+xml;base64,${btoa(`
    <svg width="120" height="80" xmlns="http://www.w3.org/2000/svg">
      <defs>
        <linearGradient id="grad1" x1="0%" y1="0%" x2="100%" y2="100%">
          <stop offset="0%" style="stop-color:#60a5fa;stop-opacity:1" />
          <stop offset="100%" style="stop-color:#3b82f6;stop-opacity:1" />
        </linearGradient>
      </defs>
      <rect width="120" height="80" fill="url(#grad1)" />
      <rect x="10" y="60" width="100" height="10" fill="rgba(255,255,255,0.3)" rx="5"/>
    </svg>
  `)}`
})
</script>

<style scoped>
.audio-clip-item {
  width: 280px;
  background: white;
  border-radius: 12px;
  padding: 16px;
  cursor: pointer;
  transition: all 0.3s ease;
  border: 2px solid transparent;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.audio-clip-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.15);
}

.audio-clip-item.selected {
  border-color: #4f46e5;
  box-shadow: 0 4px 16px rgba(79, 70, 229, 0.2);
}

.clip-thumbnail {
  position: relative;
  width: 100%;
  height: 120px;
  border-radius: 8px;
  overflow: hidden;
  margin-bottom: 12px;
}

.clip-thumbnail img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.duration-overlay {
  position: absolute;
  bottom: 8px;
  left: 8px;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 500;
}

.total-duration {
  position: absolute;
  top: 8px;
  right: 8px;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 500;
}

.clip-info {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.clip-title {
  margin: 0;
  font-size: 16px;
  font-weight: 600;
  color: #1f2328;
  line-height: 1.4;
}

.clip-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}
</style>