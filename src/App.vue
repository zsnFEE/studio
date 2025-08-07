<template>
  <div class="audio-editor">
    <!-- 顶部标题栏 -->
    <div class="header">
      <div class="title">内容名称</div>
      <t-button theme="primary" class="confirm-btn">确定</t-button>
    </div>

    <!-- 导航标签页 -->
    <div class="nav-tabs">
      <t-tabs v-model="activeTab" theme="normal">
        <t-tab-panel value="scene" label="声景"></t-tab-panel>
        <t-tab-panel value="sound" label="声音"></t-tab-panel>
        <t-tab-panel value="melody" label="旋律"></t-tab-panel>
      </t-tabs>
      
      <div class="right-tabs">
        <t-input 
          v-model="searchValue" 
          placeholder="Search" 
          class="search-input"
          prefix-icon="search"
        />
        <div class="tab-item">播放器</div>
        <div class="tab-item active">音频参数</div>
      </div>
    </div>

    <!-- 音频片段区域 -->
    <div class="audio-clips-section">
      <AudioClipItem 
        v-for="clip in audioClips" 
        :key="clip.id"
        :clip="clip"
        :isSelected="clip.id === selectedClip"
        @select="selectClip"
      />
    </div>

    <!-- 装饰点效果 -->
    <div class="decoration-dots">
      <div class="dot dot-1"></div>
      <div class="dot dot-2"></div>
      <div class="dot dot-3"></div>
      <div class="dot dot-4"></div>
      <div class="dot dot-5"></div>
      <div class="dot dot-6"></div>
      <div class="dot dot-7"></div>
    </div>

    <!-- 时间轴和播放控制 -->
    <div class="timeline-section">
      <TimelineControls />
    </div>

    <!-- 多轨道波形显示 -->
    <div class="waveform-section">
      <WaveformTracks />
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import AudioClipItem from './components/AudioClipItem.vue'
import TimelineControls from './components/TimelineControls.vue'
import WaveformTracks from './components/WaveformTracks.vue'

const activeTab = ref('sound')
const searchValue = ref('')
const selectedClip = ref(1)

const audioClips = ref([
  {
    id: 1,
    title: '星辰大海',
    duration: '0:23',
    totalDuration: '3:03',
    thumbnail: '/api/placeholder/60/40',
    tags: ['小雨', '小雨', '小雨']
  },
  {
    id: 2,
    title: '星辰大海',
    duration: '3:03',
    totalDuration: '3:03',
    thumbnail: '/api/placeholder/60/40',
    tags: ['小雨', '小雨', '小雨', '小雨']
  },
  {
    id: 3,
    title: '星辰大海',
    duration: '3:03',
    totalDuration: '3:03',
    thumbnail: '/api/placeholder/60/40',
    tags: ['小雨', '小雨', '小雨']
  },
  {
    id: 4,
    title: '星辰大海',
    duration: '3:03',
    totalDuration: '3:03',
    thumbnail: '/api/placeholder/60/40',
    tags: ['小雨', '小雨', '小雨']
  }
])

const selectClip = (clipId) => {
  selectedClip.value = clipId
}
</script>

<style scoped>
.audio-editor {
  width: 100%;
  height: 100vh;
  background: #f5f6fa;
  position: relative;
  overflow: hidden;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 24px;
  background: white;
  border-bottom: 1px solid #e7e7e7;
}

.title {
  font-size: 20px;
  font-weight: 600;
  color: #1f2328;
}

.confirm-btn {
  background: #4f46e5;
  border: none;
  border-radius: 8px;
  padding: 8px 24px;
}

.nav-tabs {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 24px;
  background: white;
  border-bottom: 1px solid #e7e7e7;
}

.right-tabs {
  display: flex;
  align-items: center;
  gap: 24px;
}

.search-input {
  width: 200px;
}

.tab-item {
  padding: 12px 0;
  color: #6b7280;
  cursor: pointer;
  border-bottom: 2px solid transparent;
  transition: all 0.3s;
}

.tab-item.active {
  color: #4f46e5;
  border-bottom-color: #4f46e5;
}

.audio-clips-section {
  display: flex;
  gap: 16px;
  padding: 24px;
  overflow-x: auto;
}

.decoration-dots {
  position: absolute;
  right: 100px;
  top: 200px;
  width: 200px;
  height: 200px;
}

.dot {
  position: absolute;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  animation: float 3s ease-in-out infinite;
}

.dot-1 {
  background: #4f46e5;
  top: 20px;
  left: 50px;
  animation-delay: 0s;
}

.dot-2 {
  background: #ef4444;
  top: 40px;
  right: 30px;
  animation-delay: 0.5s;
}

.dot-3 {
  background: #10b981;
  top: 80px;
  left: 20px;
  animation-delay: 1s;
}

.dot-4 {
  background: #f59e0b;
  bottom: 80px;
  left: 40px;
  animation-delay: 1.5s;
}

.dot-5 {
  background: #3b82f6;
  bottom: 50px;
  right: 50px;
  animation-delay: 2s;
}

.dot-6 {
  background: #8b5cf6;
  bottom: 20px;
  left: 80px;
  animation-delay: 2.5s;
}

.dot-7 {
  background: #06b6d4;
  top: 60px;
  right: 70px;
  animation-delay: 3s;
}

@keyframes float {
  0%, 100% {
    transform: translateY(0px);
    opacity: 0.7;
  }
  50% {
    transform: translateY(-10px);
    opacity: 1;
  }
}

.timeline-section {
  background: white;
  border-top: 1px solid #e7e7e7;
}

.waveform-section {
  flex: 1;
  background: white;
  border-top: 1px solid #e7e7e7;
}
</style>