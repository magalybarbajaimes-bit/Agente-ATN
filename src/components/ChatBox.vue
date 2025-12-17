import logoMTC from '../assets/MTCenter_logo.png'
<template>

  <div class="chat-wrapper">
    <!-- Botones de acceso rapido -->
    <div class="quick-sidebar">
      <div class="sidebar-header">
        <h3>Accesos Rápidos</h3>
        <p>Soluciones inmediatas</p>
      </div>
      <div class="sidebar-actions">
        <button class="sidebar-btn" @click="quickAction('MTCenter')">
          <div class="btn-icon">📄</div>
          <span>MTCenter</span>
        </button>
        <button class="sidebar-btn" @click="quickAction('ubicación')">
          <div class="btn-icon">📍</div>
          <span>Ubicación</span>
        </button>
        <button class="sidebar-btn" @click="quickAction('registro')">
          <div class="btn-icon">💲</div>
          <span>Vender recargas</span>
        </button>
        <button class="sidebar-btn" @click="quickAction('bancos')">
          <div class="btn-icon">🏦</div>
          <span>Bancos disponibles</span>
        </button>
        <button class="sidebar-btn" @click="quickAction('depósitos')">
          <div class="btn-icon">🛜</div>
          <span>Depósitos</span>
        </button>
        <button class="sidebar-btn" @click="quickAction('LiberaciónD')">
          <div class="btn-icon">🔍</div>
          <span>Liberación de depósitos</span>
        </button>
        <button class="sidebar-btn" @click="quickAction('abonos')">
          <div class="btn-icon">💳</div>
          <span>Abono de saldo</span>
        </button>
      </div>
    </div>
    <!-- CHATBOX -->
    <div class="chat-shell">
      <!-- Header -->
      <header class="chat-top colored-header">
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
          class="bubble-row">
          <!-- BOT -->
          <div v-if="msg.sender === 'bot'" class="row row-bot">
            <div class="bubble-avatar bot-logo">
              <!-- NUEVO LOGO -->
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
          v-model="userInput" type="text"
          placeholder="Describe el problema…" :disabled="loading"/>
        <button type="submit" :disabled="loading || !userInput.trim()" class="colored-send-btn">Enviar</button>
      </form>
    </div>
  </div>
</template>

<script setup>
import { ref, onUpdated, onMounted, watch } from 'vue'
// IMPORTS
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
const userInput = ref('')
const loading = ref(false)
const messagesRef = ref(null)

/* sessionId */
const sessionId = (() => {
  const saved = localStorage.getItem(LS_KEYS.session)
  if (saved) return saved
  const sid = `web-${crypto.randomUUID?.() ?? Math.random().toString(36).slice(2)}`
  localStorage.setItem(LS_KEYS.session, sid)
  return sid
})()

//Función para acciones rápidas
const quickAction = (action) => {
  const actions = {
    'MTCenter': '¿Quién es MTCenter y que hace?',
    'ubicación': '¿Dónde se encuentran?',
    'registro': '¿Cómo puedo registrarme en MTCenter para vender recargas?',
    'bancos': '¿En qué banco puedo realizar mi depósito?',
    'depósitos': '¿Dónde puedo depositar?',
    'LiberaciónD': '¿Qué horario de liberación de depósitos manejan?',
    'abonos': '¿Cómo realizo un abono de saldo en mi plataforma?'
  }
  userInput.value = actions[action]
  sendMessage()
}

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

// --- formato de texto del agente (con sub-lista Cuenta/CLABE)
const renderAgent = (raw = '') => {
  if (!raw) return ''

  // 1) Normaliza texto (soporta \n reales y \\n)
  let text = String(raw)

  // si llega doble escapado
  text = text.replace(/\\n/g, '\n')

  // quita \ al final de línea (soft break estilo markdown) y líneas que solo son "\"
  text = text
    .replace(/\r\n/g, '\n')
    .replace(/\\\s*\n/g, '\n')
    .replace(/^\s*\\\s*$/gm, '')

  // links + negritas
  text = text
    .replace(/\[([^\]]+)\]\((https?:\/\/[^\s)]+)\)/g, '<a href="$2" target="_blank" rel="noopener">$1</a>')
    .replace(/(?<!["'])\b(https?:\/\/[^\s<]+)\b/g, '<a href="$1" target="_blank" rel="noopener">$1</a>')
    .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')
    .trim()

  const lines = text.split('\n').map(l => l.trim())

  // helpers
  const isBullet = (l) => /^([•\-*])\s+/.test(l)
  const cleanBullet = (l) => l.replace(/^([•\-*])\s+/, '').trim()
  const isOrdered = (l) => /^(\d+[\.\)]|Paso\s+\d+[:\.])\s+/i.test(l)
  const cleanOrdered = (l) => l.replace(/^(\d+[\.\)]|Paso\s+\d+[:\.])\s+/i, '').trim()
  const isSubField = (l) => /^(Cuenta|CLABE)\s*:/i.test(l)

  let html = ''
  let mode = null // null | 'ul' | 'ol' | 'p'
  let buf = []

  // para UL con sublistas
  let ulItems = [] // { text: string, sub: string[] }

  const flush = () => {
    if (!mode) return
    if (mode === 'ul') {
      // vuelca ulItems si están; si no, usa buf plano
      if (ulItems.length) {
        html += `<ul class="agent-ul">` + ulItems.map(it => {
          const sub = it.sub?.length
            ? `<ul class="agent-subul">${it.sub.map(s => `<li>${s}</li>`).join('')}</ul>`
            : ''
          return `<li>${it.text}${sub}</li>`
        }).join('') + `</ul>`
      } else if (buf.length) {
        html += `<ul class="agent-ul">${buf.map(li => `<li>${li}</li>`).join('')}</ul>`
      }
      ulItems = []
      buf = []
      mode = null
      return
    }
    if (mode === 'ol') {
      if (buf.length) html += `<ol class="agent-ol">${buf.map(li => `<li>${li}</li>`).join('')}</ol>`
      buf = []
      mode = null
      return
    }
    // párrafo
    if (buf.length) html += `<p>${buf.join('<br/>')}</p>`
    buf = []
    mode = null
  }
  for (let i = 0; i < lines.length; i++) {
    const line = lines[i]
    if (!line) {
      // separador de párrafo
      if (mode === 'ul' || mode === 'ol') continue
      flush()
      continue
    }
    // BULLETS
    if (isBullet(line)) {
      const item = cleanBullet(line)
      if (mode !== 'ul') {
        flush()
        mode = 'ul'
      }
      //regla: si es "Cuenta:" o "CLABE:" se vuelve subitem del último banco
      if (isSubField(item) && ulItems.length) {
        ulItems[ulItems.length - 1].sub.push(item)
      } else {
        ulItems.push({ text: item, sub: [] })
      }
      continue
    }
    // ORDERED
    if (isOrdered(line)) {
      const item = cleanOrdered(line)
      if (mode !== 'ol') {
        flush()
        mode = 'ol'
      }
      buf.push(item)
      continue
    }
    // TEXTO NORMAL
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
        console.warn('Respuesta no OK:', res.status, text)
        await backoff(i)
        continue
      }
      const data = await res.json().catch((e) => {
        console.error('Error al parsear JSON:', e)
        return {}
      })
      console.log('JSON de la API:', data)
      const reply = (data?.reply ?? '').toString().trim()
      if (reply) return reply
    } catch (e) {
      console.error('Error de fetch intento', i, e)
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
.chat-wrapper {
  display: flex;
  max-width: 1200px; margin: 1.25rem auto;
  gap: 1.5rem; height: 90vh;
}

.quick-sidebar {
  width: 250px;
  background: #ffffff; border-radius: 1rem;
  padding: 1rem;
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.178);
}
.sidebar-header { margin-bottom: 1.5rem;}
.sidebar-header h3 {
  color: black;
  margin: 0 0 0.25rem 0;
  font-size: 1.5rem; font-weight: 600;
}
.sidebar-header p {
  color: #6c6c6c;
  margin: 0;
  font-size: 0.9rem;
}
.sidebar-actions {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
.sidebar-btn {
  background: #fdfdfd;
  border: none; border-radius: 0.75rem;
  padding: 0.75rem;
  cursor: pointer; transition: all 0.2s ease;
  display: flex; align-items: center;
  gap: 0.75rem;
  color: black; text-align: left;
}
.sidebar-btn:hover { background: #d2d2d2; transform: translate(5px);}
.btn-icon {
  width: 32px; height: 32px;
  border-radius: 0.5rem; display: flex;
  align-items: center; justify-content: center;
  font-size: 0.9rem; flex-shrink: 0;
}
.sider-btn span { font-size: 0.85rem; font-weight: 500;}

body {
  margin: 0; background: #d8d8d8be;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}

.bot-logo img {
  width: 28px; height: 28px;
  object-fit: contain;
  background: white;
}
/* General */
.chat-shell {
  flex: 1;
  max-width: 820px;
  background: #fff; border: 1px solid #c1c1c1;
  border-radius: 1.1rem;
  display: flex; flex-direction: column;
  height: 100%;
  overflow: hidden;
  box-shadow: 0 18px 40px rgba(15,23,42,.06);
}
/* Header */
.chat-top {
  display: flex; justify-content: space-between;
  align-items: center;
  padding: .9rem 1.1rem;
  border-bottom: 1px solid #edf1f4; background: #fff;
}
/* Header con color */
.colored-header {
  background: linear-gradient(90deg,rgba(1, 109, 145, 0.81) 35%, rgb(111, 212, 225) 100%) !important;
  color: white;
}
.colored-header .agent-text h2 { color: white !important;}
.colored-header .agent-text p { color: rgba(255, 255, 255, 0.8) !important;}
.colored-header .btn-ghost {
  background: rgba(157, 157, 157, 0.400);
  border: 1px solid rgba(255, 255, 255, 0.400);
  color: rgb(255, 255, 255);
}
.colored-header .btn-ghost:hover { background: rgba(255, 255, 255, 0.25);}
.colored-header .status-pill {
  background: rgba(157, 157, 157, 0.400);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.485);
}
/* Agente */
.agent-text { display: flex; flex-direction: column; }
.agent-info { display: flex; gap: .8rem; align-items: center; }
/* logo */
.brand-logo {
  width: 34px; height: 34px;
  object-fit: contain; margin-right: .23rem;
  background: white; border-radius: 50%;
}
.agent-avatar {
  width: 36px; height: 36px; border-radius: 999px;
  background: #eff6ff; border: 1px solid #dbeafe;
  display: grid; place-items: center; color: #1d4ed8; font-weight: 700; font-size: .75rem;
}
.agent-info h2 { margin: 0; font-size: 1.05rem; line-height: 1.1;}
.agent-info p { margin: 0; font-size: .74rem; color: #6b7280; }
/* Acciones header */
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
  background: #ffffff; display: grid; place-items: center;
  color: #c0c0c0; font-weight: 700; font-size: .62rem;
}
/* Burbuja bot */
.bubble-bot {
  background: #fff; border: 1px solid rgba(226,232,240,.9);
  border-radius: .9rem; border-top-left-radius: .35rem; padding: .85rem 1rem; box-shadow: 0 2px 8px rgba(0,0,0,.03);
  max-width: 78%; text-align: left;
}
.bot-content { color: #0f172a; white-space: normal; }
.bot-content p { margin: 0 0 .55rem 0; }
.bot-content a { color: var(--mt-blue); text-decoration: underline; }
.agent-ul, .agent-ol { margin: .35rem 0 .8rem 1.1rem; padding-left: 1rem; }
/* Burbuja user */
.bubble-user {
  background: var(--user-bg); color: #ffffff;
  padding: .75rem 1rem;
  border-radius: .9rem; border-bottom-right-radius: .35rem;
  box-shadow: 0 6px 16px rgba(2,6,23,.15);
  max-width: 65%; text-align: left;
}
.user-text { color:#0f172a; margin: 0; white-space: pre-wrap; }
/* Animación de “escribiendo” */
.typing-container { margin-top: .3rem; }
.typing { display: inline-flex; gap: .35rem; align-items: center; min-height: 20px; padding: .6rem .9rem; }
.dot {
  width: 7px; height: 7px;
  background: #9ca3af; border-radius: 999px;
  animation: blink 1.1s infinite ease-in-out;
}
.dot:nth-child(2){animation-delay:.2s;}
.dot:nth-child(3){animation-delay:.4s;}
@keyframes blink {0%,100%{opacity:.2;}50%{opacity:1;}}
/* Input */
.chat-input {
  display: flex; gap: .6rem; padding: .8rem 1rem;
  border-top: 1px solid #edf1f4; background: #fff;
}
.chat-input input {
  flex: 1; background: #fff; color: #111;
  border: 1px solid #d1d5db; border-radius: 999px; padding: .5rem .85rem; font-size: .88rem;
}
.chat-input input:focus {
  outline: none; border-color:#2563eb; box-shadow: 0 0 0 3px rgba(37,99,235,.12);
}
.chat-input button:disabled { opacity: .5; cursor: not-allowed; }
.colored-send-btn {
  background: linear-gradient(90deg,rgba(1, 109, 145, 0.81) 35%, rgb(111, 212, 225) 100%) !important;
  color: white !important; border: none !important; border-radius: 999px;
  padding: .5rem 1.25rem;  font-weight: 600;
  cursor: pointer; transition: all 0.2s ease;
}
.colored-send-btn:hover:not(:disabled) {
  transform: translateY(-1px); box-shadow: 0 4px 12px rgba(37, 99, 235, 0.3);
}
.colored-send-btn:disabled { 
  opacity: .5; 
  cursor: not-allowed; transform: none !important;
}

.agent-ul, .agent-ol {
  margin: .35rem 0 .8rem 1.1rem;
  padding-left: 1rem;
}

.agent-subul{
  margin: .35rem 0 0.2rem 1rem;   /* indentación extra */
  padding-left: 1.1rem;
  list-style: circle;             /* se nota que es subnivel */
}
</style>