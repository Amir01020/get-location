<template>
    <div class="app-container">
      <h1>Тест голосовых сообщений</h1>
      
      <VoiceChat ref="chatRef" />
      
      <!-- Кнопки для тестирования -->
      <div class="test-controls">
        <button @click="addTestTextMessage" class="test-btn">
          Добавить текстовое сообщение
        </button>
        <button @click="addTestVoiceMessage" class="test-btn">
          Добавить голосовое сообщение (входящее)
        </button>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref } from 'vue';
  import VoiceChat from './components/VoiceChat.vue';
  
  const chatRef = ref(null);
  
  const addTestTextMessage = () => {
    if (chatRef.value) {
      chatRef.value.addIncomingMessage('text', {
        text: 'Тестовое входящее текстовое сообщение'
      });
    }
  };
  
  const addTestVoiceMessage = () => {
    // Для демо можно использовать любой аудио URL
    if (chatRef.value) {
      chatRef.value.addIncomingMessage('voice', {
        audioUrl: 'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3',
        duration: 30
      });
    }
  };
  </script>
  
  <style scoped>
  .app-container {
    max-width: 800px;
    margin: 0 auto;
    padding: 20px;
  }
  
  h1 {
    text-align: center;
    color: #374151;
    margin-bottom: 20px;
  }
  
  .test-controls {
    margin-top: 20px;
    display: flex;
    gap: 12px;
    justify-content: center;
  }
  
  .test-btn {
    padding: 10px 20px;
    background: #3390ec;
    color: white;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    font-size: 14px;
    transition: all 0.2s;
  }
  
  .test-btn:hover {
    background: #2b7cd3;
    transform: translateY(-1px);
  }
  </style>