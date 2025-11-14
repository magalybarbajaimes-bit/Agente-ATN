<template>
  <div class="chat-shell">
    <!-- Header -->
    <header class="chat-top">
      <div class="agent-info">
        <!-- logo de la empresa -->
        <img class="brand-logo" :src="logoMTC" alt="MTCenter" />
        <div class="agent-text">
          <h2 style="text-align: left;">Soporte Técnico MTCenter</h2>
          <p>Resuelve dudas de facturación, instalaciones y servicios.</p>
        </div>
      </div>

      <!-- acciones a la derecha -->
      <div class="top-actions">
        <button class="btn-ghost" title="Borrar chat" @click="clearChat">
          Borrar
        </button>
        <span class="status-pill">En línea</span>
      </div>
    </header>

    <!-- Mensajes -->
    <main class="chat-body" ref="messagesRef">
      <div
        v-for="(msg, index) in messages"
        :key="index"
        class="bubble-row"
      >
        <!-- BOT -->
        <div v-if="msg.sender === 'bot'" class="row row-bot">
          <div class="bubble-avatar bot-logo">
            <!-- AQUÍ USAMOS EL NUEVO LOGO DEL BOT -->
            <img :src="botLogo" alt="Bot" />
          </div>
          <div class="bubble bubble-bot">
            <div class="bot-content" v-html="renderAgent(msg.text)"></div>
          </div>
        </div>

        <!-- USER -->
        <div v-else class="row row-user">
          <div class="bubble bubble-user">
            <p class="user-text">{{ msg.text }}</p>
          </div>
        </div>
      </div>

      <!-- ANIMACIÓN DE “ESCRIBIENDO...” -->
      <div v-if="loading" class="row row-bot typing-container">
        <div class="bubble-avatar bot-logo">
          <!-- Puedes usar el mismo logo del bot o uno distinto -->
          <img :src="botLogo" alt="Bot escribiendo" />
        </div>
        <div class="bubble bubble-bot typing">
          <span class="dot"></span><span class="dot"></span><span class="dot"></span>
        </div>
      </div>
    </main>

    <!-- INPUT -->
    <form class="chat-input" @submit.prevent="sendMessage">
      <input
        v-model="userInput"
        type="text"
        placeholder="Describe el problema…"
        :disabled="loading"
      />
      <button type="submit" :disabled="loading || !userInput.trim()">Enviar</button>
    </form>
  </div>
</template>

<script setup>
import { ref, onUpdated, onMounted, watch } from 'vue'
// IMPORTS CORRECTOS DESDE src/assets
import logoMTC from '../assets/MTCenter_logo.png'
import botLogo from '../assets/logo.png'

/* Claves de almacenamiento local */
const LS_KEYS = {
  messages: 'mtc_chat_messages',
  draft: 'mtc_chat_draft',
  session: 'mtc_chat_session'
}

const messages = ref([
  { sender: 'bot', text: 'Hola 👋 soy tu agente de soporte. Cuéntame tu problema.' }
])
const userInput   = ref('')
const loading     = ref(false)
const messagesRef = ref(null)

/* sessionId: usa uno guardado si existe, si no genera y guarda */
const sessionId   = (() => {
  const saved = localStorage.getItem(LS_KEYS.session)
  if (saved) return saved
  const sid = `web-${crypto.randomUUID?.() ?? Math.random().toString(36).slice(2)}`
  localStorage.setItem(LS_KEYS.session, sid)
  return sid
})()

onUpdated(() => {
  if (messagesRef.value) messagesRef.value.scrollTop = messagesRef.value.scrollHeight
})

/* Persistencia */
onMounted(() => {
  // cargar historial
  try {
    const raw = localStorage.getItem(LS_KEYS.messages)
    if (raw) {
      const parsed = JSON.parse(raw)
      if (Array.isArray(parsed) && parsed.length) {
        messages.value = parsed
      }
    }
  } catch {}
  // cargar borrador
  const draft = localStorage.getItem(LS_KEYS.draft)
  if (draft) userInput.value = draft
})

watch(messages, (val) => {
  try {
    localStorage.setItem(LS_KEYS.messages, JSON.stringify(val))
  } catch {}
}, { deep: true })

watch(userInput, (val) => {
  localStorage.setItem(LS_KEYS.draft, val ?? '')
})

/* función para borrar chat */
const clearChat = () => {
  messages.value = [
    { sender: 'bot', text: 'Hola 👋 soy tu agente de soporte. Cuéntame tu problema.' }
  ]
  userInput.value = ''
  localStorage.removeItem(LS_KEYS.messages)
  localStorage.removeItem(LS_KEYS.draft)
  // sessionId se mantiene
}

// --- formato de texto del agente (markdown simplificado)
const renderAgent = (raw = '') => {
  if (!raw) return ''
  let text = String(raw).replace(/\\n/g, '\n').trim()
    .replace(/\[([^\]]+)\]\((https?:\/\/[^\s)]+)\)/g, '<a href="$2" target="_blank">$1</a>')
    .replace(/(?<!["'])\b(https?:\/\/[^\s<]+)\b/g, '<a href="$1" target="_blank">$1</a>')
    .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')

  const lines = text.split('\n')
  let html = ''
  let mode = null
  let buf = []

  const flush = () => {
    if (!buf.length && !mode) return
    if (mode === 'ul') {
      html += `<ul class="agent-ul">${buf.map(li => `<li>${li}</li>`).join('')}</ul>`
    } else if (mode === 'ol') {
      html += `<ol class="agent-ol">${buf.map(li => `<li>${li}</li>`).join('')}</ol>`
    } else {
      html += `<p>${buf.join('<br/>')}</p>`
    }
    buf = []
    mode = null
  }

  const asBullet = l => l.replace(/^[-•]\s+/, '').trim()
  const asNum    = l => l.replace(/^(\d+[\.\)]|Paso\s+\d+[:\.])\s+/i, '').trim()

  for (let i = 0; i < lines.length; i++) {
    const rawLine = lines[i]
    const line = rawLine.trim()

    if (!line) {
      if (mode === 'ul' || mode === 'ol') continue
      flush()
      continue
    }

    if (/^[-•]\s+/.test(line)) {
      const item = asBullet(line)
      if (mode !== 'ul') { flush(); mode = 'ul' }
      buf.push(item)
      continue
    }

    if (/^(\d+[\.\)]|Paso\s+\d+[:\.])\s+/i.test(line)) {
      const item = asNum(line)
      if (mode !== 'ol') { flush(); mode = 'ol' }
      buf.push(item)
      continue
    }

    if (mode === 'ul' || mode === 'ol') flush()
    mode = mode || 'p'
    buf.push(line)
  }

  flush()
  return html
}

/* reintentos silenciosos  */
const fetchWithRetry = async (payload, tries = 3) => {
  const backoff = (i) => new Promise(r => setTimeout(r, 600 * i))

  for (let i = 1; i <= tries; i++) {
    try {
      const res = await fetch(
        'https://uh6t6ss2x1.execute-api.us-east-1.amazonaws.com/support-chat',
        {
          method: 'POST',
          headers: { 'content-type': 'application/json' },
          body: JSON.stringify(payload)
        }
      )

      if (!res.ok) {
        const text = await res.text()
        console.warn('⚠️ Respuesta no OK:', res.status, text)
        await backoff(i)
        continue
      }

      const data = await res.json().catch((e) => {
        console.error('❌ Error al parsear JSON:', e)
        return {}
      })

      console.log('🟢 JSON de la API:', data)

      const reply = (data?.reply ?? '').toString().trim()
      if (reply) return reply
    } catch (e) {
      console.error('❌ Error de fetch intento', i, e)
    }

    await backoff(i)
  }

  return 'Hubo un problema al conectar con el asistente. Intenta de nuevo.'
}

const sendMessage = async () => {
  const text = userInput.value.trim()
  if (!text) return

  messages.value.push({ sender: 'user', text })
  userInput.value = ''
  loading.value = true

  try {
    const reply = await fetchWithRetry({ message: text, sessionId }, 3)
    messages.value.push({ sender: 'bot', text: reply })
  } catch (e) {
    console.error(e)
    messages.value.push({ sender: 'bot', text: 'Hubo un problema de conexión. Intenta de nuevo.' })
  } finally {
    loading.value = false
  }
}
</script>

<style scoped>
:root {
  --mt-blue: #2563eb;
  --mt-bg: #f9fafb;
  --mt-bg: #eef2ff;
  --mt-border: #e5e7eb;
  --user-bg: #1f2937;
}

body {
  margin: 0;
  background: #e3e3e3;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}

.bot-logo img {
  width: 28px;
  height: 28px;
  object-fit: contain;
}

/* General */
.chat-shell {
  max-width: 820px;
  margin: 1.25rem auto;
  background: #fff;
  border: 1px solid var(--mt-border);
  border-radius: 1.1rem;
  display: flex;
  flex-direction: column;
  height: 90vh;
  overflow: hidden;
  box-shadow: 0 18px 40px rgba(15,23,42,.06);
}

/* Header */
.chat-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: .9rem 1.1rem;
  border-bottom: 1px solid #edf1f4;
  background: #fff;
}

.agent-text {
  display: flex;
  flex-direction: column;
}

.agent-info { display: flex; gap: .8rem; align-items: center; }

/* logo */
.brand-logo {
  width: 34px;
  height: 34px;
  object-fit: contain;
  margin-right: .23rem;
}

.agent-avatar {
  width: 36px; height: 36px;
  border-radius: 999px;
  background: #eff6ff;
  border: 1px solid #dbeafe;
  display: grid; place-items: center;
  color: #1d4ed8; font-weight: 700; font-size: .75rem;
}
.agent-info h2 { margin: 0; font-size: 1.05rem; line-height: 1.1;}
.agent-info p { margin: 0; font-size: .74rem; color: #6b7280; }

/* NUEVO: acciones header */
.top-actions { display:flex; align-items:center; gap:.5rem; }
.btn-ghost {
  background: transparent; border: 1px solid #e5e7eb; color:#374151;
  padding:.35rem .6rem; border-radius:999px; cursor:pointer; font-size:.8rem;
}
.btn-ghost:hover { background:#f3f4f6; }
.status-pill { background: #dcfce7; color: #166534; font-size: .68rem; padding: .22rem .55rem; border-radius: 999px; }

/* Body */
.chat-body { flex: 1; background: var(--mt-bg); padding: 1rem; overflow-y: auto; }
.row { display: flex; gap: .6rem; margin-bottom: .75rem; }
.row-bot { justify-content: flex-start; align-items: flex-start; }
.row-user { justify-content: flex-end; }

/* Avatar */
.bubble-avatar {
  width: 32px; height: 32px; border-radius: 999px;
  background: #eaeaea; display: grid; place-items: center;
  color: #c0c0c0; font-weight: 700; font-size: .62rem;
}

/* Burbuja bot */
.bubble-bot {
  background: #fff; border: 1px solid rgba(226,232,240,.9);
  border-radius: .9rem; border-top-left-radius: .35rem;
  padding: .85rem 1rem; box-shadow: 0 2px 8px rgba(0,0,0,.03);
  max-width: 78%; text-align: left;
}
.bot-content { color: #0f172a; white-space: normal; }
.bot-content p { margin: 0 0 .55rem 0; }
.bot-content a { color: var(--mt-blue); text-decoration: underline; }
.agent-ul, .agent-ol { margin: .35rem 0 .8rem 1.1rem; padding-left: 1rem; }

/* Burbuja user */
.bubble-user {
  background: var(--user-bg);
  color: #ffffff;
  padding: .75rem 1rem;
  border-radius: .9rem;
  border-bottom-right-radius: .35rem;
  box-shadow: 0 6px 16px rgba(2,6,23,.15);
  max-width: 65%;
  text-align: left;
}
.user-text {
  color:#0f172a;
  margin: 0; white-space: pre-wrap; 
}

/* Animación de “escribiendo” */
.typing-container { margin-top: .3rem; }
.typing { display: inline-flex; gap: .35rem; align-items: center; min-height: 20px; padding: .6rem .9rem; }
.dot {
  width: 7px; height: 7px;
  background: #9ca3af;
  border-radius: 999px;
  animation: blink 1.1s infinite ease-in-out;
}
.dot:nth-child(2){animation-delay:.2s;}
.dot:nth-child(3){animation-delay:.4s;}
@keyframes blink {0%,100%{opacity:.2;}50%{opacity:1;}}

/* Input */
.chat-input {
  display: flex; gap: .6rem;
  padding: .8rem 1rem;
  border-top: 1px solid #edf1f4;
  background: #fff;
}
.chat-input input {
  flex: 1; background: #fff; color: #111;
  border: 1px solid #d1d5db; border-radius: 999px;
  padding: .5rem .85rem; font-size: .88rem;
}
.chat-input input:focus {
  outline: none; border-color: var(--mt-blue);
  box-shadow: 0 0 0 3px rgba(37,99,235,.12);
}
.chat-input button {
  background: var(--mt-blue); 
  color: #000000;
  border-color: #000000;
  border: 1; border-radius: 999px;
  padding: .5rem 1.25rem; font-weight: 600;
}
.chat-input button:disabled { opacity: .5; cursor: not-allowed; }
</style>