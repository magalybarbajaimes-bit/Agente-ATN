<template>
  <div class="chat-shell">
    <!-- Header -->
    <header class="chat-top">
      <div class="agent-info">
        <div class="agent-avatar">MT</div>
        <div>
          <h2>Soporte técnico MTCenter</h2>
          <p>Resuelve dudas de facturación, instalaciones y servicios.</p>
        </div>
      </div>
      <span class="status-pill">En línea</span>
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
          <div class="bubble-avatar"><span>MT</span></div>
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
        <div class="bubble-avatar"><span>MT</span></div>
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
import { ref, onUpdated } from 'vue'

const messages = ref([
  { sender: 'bot', text: 'Hola 👋 soy tu agente de soporte. Cuéntame tu problema.' }
])
const userInput   = ref('')
const loading     = ref(false)
const messagesRef = ref(null)
const sessionId   = `web-${crypto.randomUUID?.() ?? Math.random().toString(36).slice(2)}`

onUpdated(() => {
  if (messagesRef.value) messagesRef.value.scrollTop = messagesRef.value.scrollHeight
})

// --- formato de texto del agente (markdown simplificado)
const renderAgent = (raw = '') => {
  if (!raw) return '';
  // normaliza saltos y aplica formateo básico
  let text = String(raw).replace(/\\n/g, '\n').trim()
    .replace(/\[([^\]]+)\]\((https?:\/\/[^\s)]+)\)/g, '<a href="$2" target="_blank">$1</a>')
    .replace(/(?<!["'])\b(https?:\/\/[^\s<]+)\b/g, '<a href="$1" target="_blank">$1</a>')
    .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>');

  const lines = text.split('\n');
  let html = '';
  let mode = null;            // null | 'ul' | 'ol' | 'p'
  let buf = [];

  const flush = () => {
    if (!buf.length && !mode) return;
    if (mode === 'ul') {
      html += `<ul class="agent-ul">${buf.map(li => `<li>${li}</li>`).join('')}</ul>`;
    } else if (mode === 'ol') {
      html += `<ol class="agent-ol">${buf.map(li => `<li>${li}</li>`).join('')}</ol>`;
    } else {
      html += `<p>${buf.join('<br/>')}</p>`;
    }
    buf = [];
    mode = null;
  };

  const asBullet = l => l.replace(/^[-•]\s+/, '').trim();
  const asNum    = l => l.replace(/^(\d+[\.\)]|Paso\s+\d+[:\.])\s+/i, '').trim();

  for (let i = 0; i < lines.length; i++) {
    const rawLine = lines[i];
    const line = rawLine.trim();

    // Línea vacía
    if (!line) {
      // Dentro de una lista no cortamos; solo ignoramos el blank
      if (mode === 'ul' || mode === 'ol') continue;
      // fuera de lista sí cerramos párrafo
      flush();
      continue;
    }

    // Bullet list
    if (/^[-•]\s+/.test(line)) {
      const item = asBullet(line);
      if (mode !== 'ul') { flush(); mode = 'ul'; }
      buf.push(item);
      continue;
    }

    // Ordered list (1. / 1) / Paso 1:)
    if (/^(\d+[\.\)]|Paso\s+\d+[:\.])\s+/i.test(line)) {
      const item = asNum(line);
      if (mode !== 'ol') { flush(); mode = 'ol'; }
      buf.push(item);
      continue;
    }

    // Texto normal
    if (mode === 'ul' || mode === 'ol') flush();
    mode = mode || 'p';
    buf.push(line);
  }

  flush();
  return html;
};


/* ========= PASO 2: reintentos silenciosos y sin fallback visible ========= */
const fetchWithRetry = async (payload, tries = 3) => {
  // opcional: timeout de 28s para no chocar con API Gateway
  const controller = new AbortController()
  const timer = setTimeout(() => controller.abort(), 28000)

  try {
    const backoff = (i) => new Promise(r => setTimeout(r, 600 * i))
    for (let i = 1; i <= tries; i++) {
      const res = await fetch(
        'https://uh6t6ss2x1.execute-api.us-east-1.amazonaws.com/support-chat',
        {
          method: 'POST',
          headers: { 'content-type': 'application/json' },
          body: JSON.stringify(payload),
          signal: controller.signal
        }
      ).catch(() => null)

      if (!res || !res.ok) { await backoff(i); continue }

      const data = await res.json().catch(() => ({}))
      const reply = (data?.reply || '').trim()

      // Si la Lambda retornó marcador “...” o vacío, reintenta
      if (reply && reply !== '...') return reply

      await backoff(i)
    }
    // Último recurso: mensaje neutro, sin “No pude…”
    return 'Estoy consultando tu información. Dame unos segundos y vuelve a intentar.'
  } finally {
    clearTimeout(timer)
  }
}

const sendMessage = async () => {
  const text = userInput.value.trim()
  if (!text) return

  messages.value.push({ sender: 'user', text })
  userInput.value = ''
  loading.value = true

  try {
    // Mantiene los puntos animados mientras reintenta
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
  --mt-border: #e5e7eb;
  --user-bg: #1f2937;
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
.agent-info { display: flex; gap: .8rem; align-items: center; }
.agent-avatar {
  width: 36px; height: 36px;
  border-radius: 999px;
  background: #eff6ff;
  border: 1px solid #dbeafe;
  display: grid; place-items: center;
  color: #1d4ed8; font-weight: 700; font-size: .75rem;
}
.agent-info h2 { margin: 0; font-size: 1rem; }
.agent-info p { margin: 0; font-size: .74rem; color: #6b7280; }
.status-pill { background: #dcfce7; color: #166534; font-size: .68rem; padding: .22rem .55rem; border-radius: 999px; }

/* Body */
.chat-body { flex: 1; background: var(--mt-bg); padding: 1rem; overflow-y: auto; }
.row { display: flex; gap: .6rem; margin-bottom: .75rem; }
.row-bot { justify-content: flex-start; align-items: flex-start; }
.row-user { justify-content: flex-end; }

/* Avatar */
.bubble-avatar {
  width: 32px; height: 32px; border-radius: 999px;
  background: #e0ebff; display: grid; place-items: center;
  color: #1d4ed8; font-weight: 700; font-size: .62rem;
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
  color: #fff;
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