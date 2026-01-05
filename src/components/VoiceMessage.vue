<template>
    <div class="voice-message" :class="{ 'voice-message-own': isOwn }">
      <button @click="togglePlay" class="play-btn">
        <svg v-if="!isPlaying" xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
          <polygon points="5 3 19 12 5 21 5 3"/>
        </svg>
        <svg v-else xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
          <rect x="6" y="4" width="4" height="16"/>
          <rect x="14" y="4" width="4" height="16"/>
        </svg>
      </button>
  
      <div class="voice-content">
        <div class="waveform-container">
          <div class="waveform-static">
            <div
              v-for="i in 30"
              :key="i"
              class="waveform-bar"
              :class="{ active: isPlaying && i <= playbackPosition }"
              :style="{ height: getBarHeight(i) + '%' }"
            ></div>
          </div>
        </div>
  
        <div class="voice-info">
          <span class="voice-duration">{{ displayTime }}</span>
        </div>
      </div>
  
      <audio
        ref="audioElement"
        :src="audioUrl"
        @loadedmetadata="onAudioLoaded"
        @timeupdate="onTimeUpdate"
        @ended="onAudioEnded"
      ></audio>
    </div>
  </template>
  
  <script setup>
  import { ref, computed, onUnmounted } from 'vue';
  
  const props = defineProps({
    audioUrl: {
      type: String,
      required: true
    },
    duration: {
      type: Number,
      default: 0
    },
    isOwn: {
      type: Boolean,
      default: false
    }
  });
  
  const audioElement = ref(null);
  const isPlaying = ref(false);
  const currentTime = ref(0);
  const totalDuration = ref(props.duration || 0);
  const playbackPosition = ref(0);
  
  // Генерируем случайные высоты для волны (имитация реальной аудиоволны)
  const waveformData = ref(
    Array(30).fill(0).map(() => Math.random() * 60 + 40)
  );
  
  const getBarHeight = (index) => {
    return waveformData.value[index - 1] || 50;
  };
  
  const displayTime = computed(() => {
    const time = isPlaying.value ? currentTime.value : totalDuration.value;
    const minutes = Math.floor(time / 60);
    const seconds = Math.floor(time % 60);
    return `${minutes}:${seconds.toString().padStart(2, '0')}`;
  });
  
  const togglePlay = () => {
    if (!audioElement.value) return;
  
    if (isPlaying.value) {
      audioElement.value.pause();
      isPlaying.value = false;
    } else {
      audioElement.value.play();
      isPlaying.value = true;
    }
  };
  
  const onAudioLoaded = () => {
    if (audioElement.value && !totalDuration.value) {
      totalDuration.value = Math.floor(audioElement.value.duration);
    }
  };
  
  const onTimeUpdate = () => {
    if (audioElement.value) {
      currentTime.value = audioElement.value.currentTime;
      
      // Вычисляем позицию воспроизведения для анимации волны
      const progress = currentTime.value / totalDuration.value;
      playbackPosition.value = Math.floor(progress * 30);
    }
  };
  
  const onAudioEnded = () => {
    isPlaying.value = false;
    currentTime.value = 0;
    playbackPosition.value = 0;
  };
  
  onUnmounted(() => {
    if (audioElement.value) {
      audioElement.value.pause();
    }
  });
  </script>
  
  <style scoped>
  .voice-message {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 8px 12px;
    background: #ffffff;
    border-radius: 12px;
    max-width: 300px;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  }
  
  .voice-message-own {
    background: #dcf8c6;
    margin-left: auto;
  }
  
  .play-btn {
    width: 36px;
    height: 36px;
    border-radius: 50%;
    background: #3390ec;
    border: none;
    color: white;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s;
    flex-shrink: 0;
  }
  
  .voice-message-own .play-btn {
    background: #34b7f1;
  }
  
  .play-btn:hover {
    transform: scale(1.05);
    background: #2b7cd3;
  }
  
  .voice-message-own .play-btn:hover {
    background: #2ca5d8;
  }
  
  .voice-content {
    display: flex;
    flex-direction: column;
    gap: 4px;
    flex: 1;
    min-width: 0;
  }
  
  .waveform-container {
    width: 100%;
    height: 24px;
    display: flex;
    align-items: center;
  }
  
  .waveform-static {
    display: flex;
    align-items: center;
    gap: 2px;
    height: 100%;
    width: 100%;
  }
  
  .waveform-bar {
    width: 3px;
    background: #d1d5db;
    border-radius: 2px;
    transition: background 0.1s ease;
  }
  
  .waveform-bar.active {
    background: #3390ec;
  }
  
  .voice-message-own .waveform-bar.active {
    background: #34b7f1;
  }
  
  .voice-info {
    display: flex;
    align-items: center;
    justify-content: flex-end;
  }
  
  .voice-duration {
    font-size: 12px;
    color: #6b7280;
    font-weight: 500;
  }
  
  audio {
    display: none;
  }
  </style>