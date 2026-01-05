<template>
    <div class="voice-record-container">
      <!-- Кнопка записи -->
      <button
        v-if="!isRecording"
        @click="startRecording"
        class="voice-record-btn"
        :disabled="isProcessing"
      >
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <path d="M12 2a3 3 0 0 0-3 3v7a3 3 0 0 0 6 0V5a3 3 0 0 0-3-3Z"/>
          <path d="M19 10v2a7 7 0 0 1-14 0v-2"/>
          <line x1="12" x2="12" y1="19" y2="22"/>
        </svg>
      </button>
  
      <!-- Интерфейс во время записи -->
      <div v-else class="recording-interface">
        <button @click="cancelRecording" class="cancel-btn">
          <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <line x1="18" x2="6" y1="6" y2="18"/>
            <line x1="6" x2="18" y1="6" y2="18"/>
          </svg>
        </button>
  
        <div class="recording-info">
          <div class="recording-animation">
            <div class="recording-dot"></div>
          </div>
          <span class="recording-time">{{ formattedTime }}</span>
          <div class="waveform">
            <div
              v-for="i in 20"
              :key="i"
              class="waveform-bar"
              :style="{ height: waveformHeights[i - 1] + '%' }"
            ></div>
          </div>
        </div>
  
        <button @click="stopRecording" class="send-btn">
          <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <line x1="22" x2="11" y1="2" y2="13"/>
            <polygon points="22 2 15 22 11 13 2 9 22 2"/>
          </svg>
        </button>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, computed, onUnmounted } from 'vue';
  
  const emit = defineEmits(['voiceRecorded']);
  
  const isRecording = ref(false);
  const isProcessing = ref(false);
  const recordingTime = ref(0);
  const mediaRecorder = ref(null);
  const audioChunks = ref([]);
  const recordingTimer = ref(null);
  const waveformHeights = ref(Array(20).fill(20));
  const waveformInterval = ref(null);
  const audioContext = ref(null);
  const analyser = ref(null);
  
  const formattedTime = computed(() => {
    const minutes = Math.floor(recordingTime.value / 60);
    const seconds = recordingTime.value % 60;
    return `${minutes}:${seconds.toString().padStart(2, '0')}`;
  });
  
  // Анимация волны
  const animateWaveform = () => {
    waveformHeights.value = waveformHeights.value.map(() => 
      Math.random() * 80 + 20
    );
  };
  
  const startRecording = async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      
      // Создаем MediaRecorder
      mediaRecorder.value = new MediaRecorder(stream);
      audioChunks.value = [];
      
      // Создаем AudioContext для визуализации
      audioContext.value = new (window.AudioContext || window.webkitAudioContext)();
      analyser.value = audioContext.value.createAnalyser();
      const source = audioContext.value.createMediaStreamSource(stream);
      source.connect(analyser.value);
      
      mediaRecorder.value.ondataavailable = (event) => {
        if (event.data.size > 0) {
          audioChunks.value.push(event.data);
        }
      };
      
      mediaRecorder.value.onstop = () => {
        const audioBlob = new Blob(audioChunks.value, { type: 'audio/webm' });
        const audioUrl = URL.createObjectURL(audioBlob);
        
        emit('voiceRecorded', {
          blob: audioBlob,
          url: audioUrl,
          duration: recordingTime.value
        });
        
        // Останавливаем все треки
        stream.getTracks().forEach(track => track.stop());
        if (audioContext.value) {
          audioContext.value.close();
        }
      };
      
      mediaRecorder.value.start();
      isRecording.value = true;
      recordingTime.value = 0;
      
      // Запускаем таймер
      recordingTimer.value = setInterval(() => {
        recordingTime.value++;
      }, 1000);
      
      // Запускаем анимацию волны
      waveformInterval.value = setInterval(animateWaveform, 100);
      
    } catch (error) {
      console.error('Ошибка доступа к микрофону:', error);
      alert('Не удалось получить доступ к микрофону');
    }
  };
  
  const stopRecording = () => {
    if (mediaRecorder.value && mediaRecorder.value.state !== 'inactive') {
      mediaRecorder.value.stop();
      clearInterval(recordingTimer.value);
      clearInterval(waveformInterval.value);
      isRecording.value = false;
      recordingTime.value = 0;
    }
  };
  
  const cancelRecording = () => {
    if (mediaRecorder.value && mediaRecorder.value.state !== 'inactive') {
      const stream = mediaRecorder.value.stream;
      mediaRecorder.value.stop();
      stream.getTracks().forEach(track => track.stop());
      
      clearInterval(recordingTimer.value);
      clearInterval(waveformInterval.value);
      isRecording.value = false;
      recordingTime.value = 0;
      audioChunks.value = [];
      
      if (audioContext.value) {
        audioContext.value.close();
      }
    }
  };
  
  onUnmounted(() => {
    if (recordingTimer.value) {
      clearInterval(recordingTimer.value);
    }
    if (waveformInterval.value) {
      clearInterval(waveformInterval.value);
    }
    if (audioContext.value) {
      audioContext.value.close();
    }
  });
  </script>
  
  <style scoped>
  .voice-record-container {
    display: flex;
    align-items: center;
    gap: 8px;
  }
  
  .voice-record-btn {
    width: 40px;
    height: 40px;
    border-radius: 50%;
    background: #3390ec;
    border: none;
    color: white;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s;
  }
  
  .voice-record-btn:hover {
    background: #2b7cd3;
    transform: scale(1.05);
  }
  
  .voice-record-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
  
  .recording-interface {
    display: flex;
    align-items: center;
    gap: 12px;
    background: #f0f2f5;
    padding: 8px 12px;
    border-radius: 20px;
    width: 100%;
    max-width: 400px;
  }
  
  .cancel-btn {
    width: 32px;
    height: 32px;
    border-radius: 50%;
    background: #e5e7eb;
    border: none;
    color: #6b7280;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s;
    flex-shrink: 0;
  }
  
  .cancel-btn:hover {
    background: #d1d5db;
  }
  
  .recording-info {
    display: flex;
    align-items: center;
    gap: 8px;
    flex: 1;
  }
  
  .recording-animation {
    display: flex;
    align-items: center;
    justify-content: center;
  }
  
  .recording-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: #ef4444;
    animation: pulse 1.5s ease-in-out infinite;
  }
  
  @keyframes pulse {
    0%, 100% {
      opacity: 1;
      transform: scale(1);
    }
    50% {
      opacity: 0.5;
      transform: scale(1.2);
    }
  }
  
  .recording-time {
    font-size: 14px;
    color: #374151;
    font-weight: 500;
    min-width: 40px;
  }
  
  .waveform {
    display: flex;
    align-items: center;
    gap: 2px;
    height: 24px;
    flex: 1;
  }
  
  .waveform-bar {
    width: 3px;
    background: #3390ec;
    border-radius: 2px;
    transition: height 0.1s ease;
  }
  
  .send-btn {
    width: 32px;
    height: 32px;
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
  
  .send-btn:hover {
    background: #2b7cd3;
    transform: scale(1.05);
  }
  </style>