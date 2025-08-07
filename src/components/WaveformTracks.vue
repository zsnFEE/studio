<template>
  <div class="waveform-tracks">
    <!-- 轨道列表 -->
    <div class="tracks-container">
      <WaveformTrack
        v-for="track in tracks"
        :key="track.id"
        :track="track"
        :isSelected="track.id === selectedTrack"
        @select="selectTrack"
        @toggle-solo="toggleSolo"
        @toggle-mute="toggleMute"
      />
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import WaveformTrack from './WaveformTrack.vue'

const selectedTrack = ref(1)

const tracks = ref([
  {
    id: 1,
    name: '星辰大海',
    type: 'S',
    color: '#e879f9',
    segments: [
      { start: 0, duration: 43, amplitude: 0.8 }
    ],
    isSolo: false,
    isMuted: false
  },
  {
    id: 2,
    name: '笛子',
    type: 'S',
    color: '#10b981',
    segments: [
      { start: 5, duration: 8, amplitude: 0.6 },
      { start: 18, duration: 8, amplitude: 0.7 },
      { start: 30, duration: 8, amplitude: 0.5 },
      { start: 42, duration: 8, amplitude: 0.8 },
      { start: 55, duration: 8, amplitude: 0.6 },
      { start: 68, duration: 8, amplitude: 0.7 },
      { start: 80, duration: 8, amplitude: 0.9 }
    ],
    isSolo: false,
    isMuted: false
  },
  {
    id: 3,
    name: '微风',
    type: 'S',
    color: '#3b82f6',
    segments: [
      { start: 10, duration: 12, amplitude: 0.4 },
      { start: 35, duration: 15, amplitude: 0.6 },
      { start: 65, duration: 18, amplitude: 0.5 },
      { start: 95, duration: 10, amplitude: 0.7 }
    ],
    isSolo: false,
    isMuted: false
  },
  {
    id: 4,
    name: '夜晚海浪',
    type: 'S',
    color: '#f59e0b',
    segments: [
      { start: 25, duration: 20, amplitude: 0.7 },
      { start: 70, duration: 25, amplitude: 0.8 }
    ],
    isSolo: false,
    isMuted: false
  },
  {
    id: 5,
    name: '篝火',
    type: 'S',
    color: '#ef4444',
    segments: [
      { start: 15, duration: 10, amplitude: 0.5 },
      { start: 50, duration: 15, amplitude: 0.6 }
    ],
    isSolo: false,
    isMuted: false
  }
])

const selectTrack = (trackId) => {
  selectedTrack.value = trackId
}

const toggleSolo = (trackId) => {
  const track = tracks.value.find(t => t.id === trackId)
  if (track) {
    track.isSolo = !track.isSolo
  }
}

const toggleMute = (trackId) => {
  const track = tracks.value.find(t => t.id === trackId)
  if (track) {
    track.isMuted = !track.isMuted
  }
}
</script>

<style scoped>
.waveform-tracks {
  flex: 1;
  background: white;
  overflow-y: auto;
}

.tracks-container {
  display: flex;
  flex-direction: column;
}
</style>