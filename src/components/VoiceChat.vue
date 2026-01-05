<template>
    <div class="voice-chat-container">
      <!-- Область сообщений -->
      <div class="messages-area" ref="messagesContainer">
        <div
          v-for="message in messages"
          :key="message.id"
          class="message-wrapper"
          :class="{ 'message-own': message.isOwn }"
        >
          <!-- Текстовое сообщение -->
          <div v-if="message.type === 'text'" class="text-message" :class="{ 'text-message-own': message.isOwn }">
            <p>{{ message.text }}</p>
            <span class="message-time">{{ formatTime(message.timestamp) }}</span>
          </div>
  
          <!-- Голосовое сообщение -->
          <VoiceMessage
            v-else-if="message.type === 'voice'"
            :audioUrl="message.audioUrl"
            :duration="message.duration"
            :isOwn="message.isOwn"
          />
        </div>
  
        <!-- Индикатор отсутствия сообщений -->
        <div v-if="messages.length === 0" class="empty-state">
          <svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" style="color: #d1d5db;">
            <path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"/>
          </svg>
          <p>Нет сообщений</p>
        </div>
      </div>
  
      <!-- Область ввода -->
      <div class="input-area">
        <div class="input-container">
          <!-- Текстовое поле -->
          <input
            v-if="!isRecording"
            v-model="messageText"
            type="text"
            placeholder="Введите сообщение..."
            class="message-input"
            @keyup.enter="sendTextMessage"
          />
  
          <!-- Кнопка отправки текста -->
          <button
            v-if="messageText.trim() && !isRecording"
            @click="sendTextMessage"
            class="send-text-btn"
          >
            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <line x1="22" x2="11" y1="2" y2="13"/>
              <polygon points="22 2 15 22 11 13 2 9 22 2"/>
            </svg>
          </button>
  
          <!-- Компонент записи голоса -->
          <VoiceRecordButton
            v-if="!messageText.trim()"
            @voiceRecorded="handleVoiceRecorded"
          />
        </div>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, nextTick, onMounted } from 'vue';
  import VoiceMessage from './VoiceMessage.vue';
  import VoiceRecordButton from './VoiceRecordButton.vue';
  
  const messages = ref([]);
  const messageText = ref('');
  const messagesContainer = ref(null);
  const isRecording = ref(false);
  let messageIdCounter = 0;
  
  // Пример начальных сообщений
  onMounted(() => {
    // Можно добавить тестовые сообщения
    // addMessage('text', { text: 'Привет! Как дела?' }, false);
  });
  
  const addMessage = (type, data, isOwn = true) => {
    const message = {
      id: ++messageIdCounter,
      type,
      isOwn,
      timestamp: new Date(),
      ...data
    };
  
    messages.value.push(message);
    scrollToBottom();
  };
  
  const sendTextMessage = () => {
    if (messageText.value.trim()) {
      addMessage('text', { text: messageText.value.trim() }, true);
      messageText.value = '';
    }
  };
  
  const handleVoiceRecorded = ({ blob, url, duration }) => {
    addMessage('voice', {
      audioUrl: url,
      duration: duration,
      blob: blob
    }, true);
  
    // Здесь можно отправить blob на сервер
    // await uploadVoiceMessage(blob);
  };
  
  const scrollToBottom = async () => {
    await nextTick();
    if (messagesContainer.value) {
      messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight;
    }
  };
  
  const formatTime = (date) => {
    const hours = date.getHours().toString().padStart(2, '0');
    const minutes = date.getMinutes().toString().padStart(2, '0');
    return `${hours}:${minutes}`;
  };
  
  // Функция для добавления входящего сообщения (для демо)
  const addIncomingMessage = (type, data) => {
    addMessage(type, data, false);
  };
  
  // Экспортируем для использования извне (опционально)
  defineExpose({
    addIncomingMessage
  });
  </script>
  
  <style scoped>
  .voice-chat-container {
    display: flex;
    flex-direction: column;
    height: 600px;
    background: #f0f2f5;
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  }
  
  .messages-area {
    flex: 1;
    overflow-y: auto;
    padding: 16px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  
  .messages-area::-webkit-scrollbar {
    width: 6px;
  }
  
  .messages-area::-webkit-scrollbar-track {
    background: transparent;
  }
  
  .messages-area::-webkit-scrollbar-thumb {
    background: #cbd5e1;
    border-radius: 3px;
  }
  
  .messages-area::-webkit-scrollbar-thumb:hover {
    background: #94a3b8;
  }
  
  .message-wrapper {
    display: flex;
    margin-bottom: 4px;
  }
  
  .message-wrapper.message-own {
    justify-content: flex-end;
  }
  
  .text-message {
    background: #ffffff;
    padding: 8px 12px;
    border-radius: 12px;
    max-width: 70%;
    word-wrap: break-word;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  }
  
  .text-message-own {
    background: #dcf8c6;
  }
  
  .text-message p {
    margin: 0 0 4px 0;
    color: #374151;
    font-size: 14px;
    line-height: 1.4;
  }
  
  .message-time {
    font-size: 11px;
    color: #6b7280;
    display: block;
    text-align: right;
  }
  
  .empty-state {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 100%;
    color: #9ca3af;
    gap: 12px;
  }
  
  .empty-state p {
    margin: 0;
    font-size: 16px;
  }
  
  .input-area {
    padding: 12px 16px;
    background: #ffffff;
    border-top: 1px solid #e5e7eb;
  }
  
  .input-container {
    display: flex;
    align-items: center;
    gap: 8px;
    background: #f9fafb;
    border-radius: 24px;
    padding: 4px 8px;
  }
  
  .message-input {
    flex: 1;
    border: none;
    background: transparent;
    padding: 8px 12px;
    font-size: 14px;
    outline: none;
    color: #374151;
  }
  
  .message-input::placeholder {
    color: #9ca3af;
  }
  
  .send-text-btn {
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
  
  .send-text-btn:hover {
    background: #2b7cd3;
    transform: scale(1.05);
  }
  </style>