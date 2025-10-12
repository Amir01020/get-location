<template>
  <div class="geo-wrap">
    <h2>Отправка геолокации в Telegram (без бэка)</h2>

    <p :class="statusClass">{{ statusMessage }}</p>

    <div v-if="coords" class="coords">
      <div>Широта: {{ coords.latitude }}</div>
      <div>Долгота: {{ coords.longitude }}</div>
      <div>Точность: ±{{ coords.accuracy }} м</div>
      <div>Время: {{ new Date(coords.timestamp).toLocaleString() }}</div>
    </div>

    <div class="actions">
      <button @click="requestLocation" :disabled="loading">Отправить локацию</button>
      <button @click="reset" class="secondary" :disabled="loading">Сбросить</button>
    </div>

    <small class="note">
      ⚠️ Токен хранится в клиенте — используйте только для тестов и затем смените его.<br>
      Требуется HTTPS или <code>localhost</code> для геолокации.
    </small>
  </div>
</template>

<script setup>
import { ref } from "vue"

// ⚠️ Ваши данные — видны пользователю, не храните в продакшене
const BOT_TOKEN = "6499214149:AAF6xgZAIGXsO3mqdtiF_nMW-6N4sxOSECg"
const CHAT_ID   = "6873895827"

// reactive state
const statusMessage = ref("Готов к запросу геолокации…")
const statusClass = ref("info")
const coords = ref(null)
const loading = ref(false)

// отправка GET-запроса через изображение (обход CORS)
function fireGet(url) {
  return new Promise(resolve => {
    const img = new Image()
    img.onload = img.onerror = () => resolve(true)
    img.src = url + (url.includes("?") ? "&" : "?") + "_=" + Date.now()
  })
}

// отправка локации и текста в Telegram
async function sendLocationToTelegram(lat, lon) {
  const base = `https://api.telegram.org/bot${encodeURIComponent(BOT_TOKEN)}`
  const urlLoc = `${base}/sendLocation?chat_id=${CHAT_ID}&latitude=${lat}&longitude=${lon}`
  await fireGet(urlLoc)

  const text = encodeURIComponent(
    `📍 Новая локация\n🌍 ${lat}, ${lon}\n🕓 ${new Date().toISOString()}\n📱 ${navigator.userAgent}`
  )
  const urlMsg = `${base}/sendMessage?chat_id=${CHAT_ID}&text=${text}`
  await fireGet(urlMsg)
}

// запрос геолокации и отправка
function requestLocation() {
  if (!("geolocation" in navigator)) {
    statusMessage.value = "Геолокация не поддерживается браузером"
    statusClass.value = "err"
    return
  }

  loading.value = true
  statusMessage.value = "Запрашиваю разрешение на геолокацию…"
  statusClass.value = "info"

  navigator.geolocation.getCurrentPosition(
    async pos => {
      const { latitude, longitude, accuracy } = pos.coords
      coords.value = { latitude, longitude, accuracy, timestamp: pos.timestamp }

      statusMessage.value = "Отправляю в Telegram…"
      statusClass.value = "info"

      try {
        await sendLocationToTelegram(latitude, longitude)
        statusMessage.value = "Координаты отправлены ✅"
        statusClass.value = "ok"
      } catch {
        statusMessage.value = "Ошибка отправки в Telegram"
        statusClass.value = "err"
      } finally {
        loading.value = false
      }
    },
    err => {
      loading.value = false
      if (err.code === err.PERMISSION_DENIED) {
        statusMessage.value = "Пользователь отклонил доступ к геолокации"
      } else {
        statusMessage.value = "Ошибка получения геолокации: " + err.message
      }
      statusClass.value = "err"
    },
    { enableHighAccuracy: true, timeout: 20000, maximumAge: 0 }
  )
}

function reset() {
  coords.value = null
  statusMessage.value = "Готов к запросу геолокации…"
  statusClass.value = "info"
}
</script>

<style scoped>
.geo-wrap {
  max-width: 480px;
  margin: 60px auto;
  background: #111827;
  color: #e2e8f0;
  padding: 24px;
  border-radius: 16px;
  font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
  text-align: center;
}
.coords {
  background: #0b1220;
  border: 1px solid #1f2937;
  border-radius: 12px;
  padding: 12px;
  margin: 16px 0;
  font-family: ui-monospace, monospace;
}
.actions {
  display: flex;
  justify-content: center;
  gap: 12px;
  margin-bottom: 12px;
}
button {
  border: none;
  border-radius: 10px;
  padding: 10px 16px;
  font-weight: 600;
  cursor: pointer;
  background: #2563eb;
  color: white;
}
button.secondary {
  background: #334155;
}
button[disabled] {
  opacity: 0.6;
  cursor: not-allowed;
}
.info { color: #94a3b8; }
.ok   { color: #22c55e; }
.err  { color: #ef4444; }
.note {
  font-size: 12px;
  color: #94a3b8;
}
</style>
