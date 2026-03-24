<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue'

const zapierScriptSrc =
  'https://interfaces.zapier.com/assets/web-components/zapier-interfaces/zapier-interfaces.esm.js'

const mirroredInput = ref('')
const mirrorMessages = ref([])
const syncStatus = ref('Sincronizando con el chat de Zapier...')

let pollTimer
const seenMirrorKeys = new Set()

function normalizeText(text) {
  return (text || '').replace(/\s+/g, ' ').trim()
}

function addMirrorMessage(role, text) {
  const normalized = normalizeText(text)
  if (!normalized) return
  const key = `${role}:${normalized}`
  if (seenMirrorKeys.has(key)) return
  seenMirrorKeys.add(key)
  mirrorMessages.value.push({ id: `${Date.now()}-${Math.random().toString(16).slice(2)}`, role, text: normalized })
}

// Brute-force scanner: inspect host.shadowRoot if available, otherwise scan host subtree
function scanZapierMessages() {
  const host = document.querySelector('zapier-interfaces-chatbot-embed')
  if (!host) {
    syncStatus.value = 'Esperando embed de Zapier...'
    return
  }

  let candidates = []
  try {
    if (host.shadowRoot) candidates = Array.from(host.shadowRoot.querySelectorAll('*'))
    else if (host.querySelectorAll) candidates = Array.from(host.querySelectorAll('*'))
    else candidates = Array.from(document.querySelectorAll('zapier-interfaces-chatbot-embed *'))
  } catch (e) {
    candidates = Array.from(document.querySelectorAll('zapier-interfaces-chatbot-embed *'))
  }

  let added = 0
  for (const el of candidates) {
    const text = normalizeText(el.innerText || el.textContent || '')
    if (!text) continue
    if (text.length < 2 || text.length > 1500) continue
    if (/powered by|type your message/i.test(text)) continue

    const attrs = `${el.getAttribute?.('data-role') || ''} ${el.getAttribute?.('data-message-author') || ''} ${el.className || ''}`.toLowerCase()
    const role = /user|you|me|outgoing/.test(attrs) ? 'user' : 'assistant'

    const before = mirrorMessages.value.length
    addMirrorMessage(role, text)
    if (mirrorMessages.value.length > before) added++
  }

  if (added > 0) syncStatus.value = `Sincronizado (${mirrorMessages.value.length} mensajes)`
}

// Simple poller
function observeChat() {
  pollTimer = setInterval(scanZapierMessages, 1500)
}

function setNativeInputValue(input, value) {
  try {
    const descriptor = Object.getOwnPropertyDescriptor(Object.getPrototypeOf(input), 'value')
    descriptor?.set?.call(input, value)
  } catch (e) {
    // ignore
  }
}

function sendToZapier() {
  const text = normalizeText(mirroredInput.value)
  if (!text) return

  addMirrorMessage('user', text)
  mirroredInput.value = ''

  const host = document.querySelector('zapier-interfaces-chatbot-embed')
  if (!host) {
    syncStatus.value = 'No se encontró el embed de Zapier para enviar el mensaje.'
    return
  }

  // brute-force: try shadowRoot inputs, then host inputs, then document-wide inputs under embed
  let inputs = []
  try {
    if (host.shadowRoot) inputs = Array.from(host.shadowRoot.querySelectorAll('textarea, input'))
    if (inputs.length === 0 && host.querySelectorAll) inputs = Array.from(host.querySelectorAll('textarea, input'))
    if (inputs.length === 0) inputs = Array.from(document.querySelectorAll('zapier-interfaces-chatbot-embed textarea, zapier-interfaces-chatbot-embed input'))
  } catch (e) {
    inputs = Array.from(document.querySelectorAll('zapier-interfaces-chatbot-embed textarea, zapier-interfaces-chatbot-embed input'))
  }

  const target = inputs.at(-1)
  if (target) {
    setNativeInputValue(target, text)
    target.dispatchEvent(new Event('input', { bubbles: true }))
    target.dispatchEvent(new Event('change', { bubbles: true }))
  }

  // try to click a send-like button
  let buttons = []
  try {
    if (host.shadowRoot) buttons = Array.from(host.shadowRoot.querySelectorAll('button, [role="button"]'))
    if (buttons.length === 0 && host.querySelectorAll) buttons = Array.from(host.querySelectorAll('button, [role="button"]'))
    if (buttons.length === 0) buttons = Array.from(document.querySelectorAll('zapier-interfaces-chatbot-embed button, zapier-interfaces-chatbot-embed [role="button"]'))
  } catch (e) {
    buttons = Array.from(document.querySelectorAll('zapier-interfaces-chatbot-embed button, zapier-interfaces-chatbot-embed [role="button"]'))
  }

  for (const b of buttons) {
    const label = `${b.textContent || ''} ${b.getAttribute?.('aria-label') || ''}`.toLowerCase()
    if (/send|enviar|submit/.test(label)) {
      try { b.click() } catch (e) {}
      syncStatus.value = 'Mensaje enviado (bruto) al chat de Zapier.'
      return
    }
  }

  // fallback: dispatch Enter on target input
  if (target) {
    try {
      target.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter', code: 'Enter', bubbles: true }))
    } catch (e) {}
    syncStatus.value = 'Mensaje reflejado (fallback Enter) al chat de Zapier.'
    return
  }

  syncStatus.value = 'Mensaje agregado al espejo, no se pudo enviar al widget.'
}

onMounted(() => {
  if (!document.querySelector(`script[src="${zapierScriptSrc}"]`)) {
    const script = document.createElement('script')
    script.type = 'module'
    script.async = true
    script.src = zapierScriptSrc
    document.head.appendChild(script)
  }

  observeChat()
  scanZapierMessages()
})

onBeforeUnmount(() => {
  clearInterval(pollTimer)
})
</script>

<template>
  <main class="dashboard">
    <section class="panel">
      <header class="panel-header">
        <h1>Chat Zapier</h1>
        <p>{{ syncStatus }}</p>
      </header>

      <form class="input-bar" @submit.prevent="sendToZapier">
        <input
          v-model="mirroredInput"
          type="text"
          placeholder="Escribe aquí para enviar al chat de Zapier"
        />
        <button type="submit">Enviar</button>
      </form>

      <div class="chat-wrap">
        <zapier-interfaces-chatbot-embed
          is-popup="false"
          chatbot-id="cmmdnvuvz003obp3j5uedphlx"
          height="600px"
          width="400px"
        />
      </div>
    </section>

    <section class="panel mirror">
      <header class="panel-header">
        <h2>Chat espejo</h2>
        <p>Replica lo escrito y lo detectado desde el widget de Zapier.</p>
      </header>

      <div class="mirror-thread">
        <p v-if="mirrorMessages.length === 0" class="empty-state">
          Aún no hay mensajes. Escribe en el campo de la izquierda para iniciar.
        </p>
        <article
          v-for="message in mirrorMessages"
          :key="message.id"
          class="bubble"
          :class="message.role"
        >
          <strong>{{ message.role === 'user' ? 'Tú' : 'Zapier' }}</strong>
          <p>{{ message.text }}</p>
        </article>
      </div>
    </section>
  </main>
</template>
