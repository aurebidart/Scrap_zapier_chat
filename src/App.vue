<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue'

const zapierScriptSrc =
  'https://interfaces.zapier.com/assets/web-components/zapier-interfaces/zapier-interfaces.esm.js'

const mirroredInput = ref('')
const mirrorMessages = ref([])
const syncStatus = ref('Sincronizando con el chat de Zapier...')

let observer
let pollTimer
let rafHandle
const seenMirrorKeys = new Set()

function normalizeText(text) {
  return text.replace(/\s+/g, ' ').trim()
}

function addMirrorMessage(role, text) {
  const normalized = normalizeText(text)
  if (!normalized) return

  const key = `${role}:${normalized}`
  if (seenMirrorKeys.has(key)) return

  seenMirrorKeys.add(key)
  mirrorMessages.value.push({
    id: `${Date.now()}-${Math.random().toString(16).slice(2)}`,
    role,
    text: normalized,
  })
}

function getAllRoots(hostElement) {
  const roots = [hostElement]
  const stack = [hostElement]

  while (stack.length > 0) {
    const current = stack.pop()

    if (current?.shadowRoot) {
      roots.push(current.shadowRoot)
      stack.push(current.shadowRoot)
    }

    const children = current?.children ?? []
    for (const child of children) {
      stack.push(child)
    }
  }

  return roots
}

function nodeBelongsToHost(node, hostElement) {
  let current = node
  while (current) {
    if (current === hostElement) return true
    const root = current.getRootNode?.()
    current = current.parentNode ?? root?.host
  }
  return false
}

function detectRoleFromNode(element) {
  const attrs = [
    element.getAttribute?.('data-role') ?? '',
    element.getAttribute?.('data-message-author') ?? '',
    element.getAttribute?.('aria-label') ?? '',
    element.className ?? '',
  ]
    .join(' ')
    .toLowerCase()

  if (/assistant|bot|ai|zapier|agent|incoming|response/.test(attrs)) {
    return 'assistant'
  }
  if (/user|human|me|you|outgoing|request/.test(attrs)) {
    return 'user'
  }
  return 'assistant'
}

function scanZapierMessages() {
  const host = document.querySelector('zapier-interfaces-chatbot-embed')
  if (!host) {
    syncStatus.value = 'Esperando a que cargue el embed de Zapier...'
    return
  }

  const roots = getAllRoots(host)
  const selector =
    '[data-message-author], [data-role], [data-testid*="message"], [class*="message"], [class*="bubble"], li, p, article'

  let newMessages = 0

  for (const root of roots) {
    const nodes = root.querySelectorAll?.(selector) ?? []
    for (const node of nodes) {
      const text = normalizeText(node.textContent ?? '')
      if (!text) continue
      if (text.length < 2 || text.length > 1200) continue
      if (/type your message|powered by|send/i.test(text)) continue

      const role = detectRoleFromNode(node)
      const sizeBefore = mirrorMessages.value.length
      addMirrorMessage(role, text)
      if (mirrorMessages.value.length > sizeBefore) {
        newMessages += 1
      }
    }
  }

  if (newMessages > 0) {
    syncStatus.value = `Sincronizado. ${mirrorMessages.value.length} mensajes detectados.`
  }
}

function scheduleScan() {
  if (rafHandle) return
  rafHandle = requestAnimationFrame(() => {
    rafHandle = null
    scanZapierMessages()
  })
}

function observeChat() {
  observer = new MutationObserver(() => {
    scheduleScan()
  })

  observer.observe(document.body, {
    childList: true,
    subtree: true,
    characterData: true,
  })
}

function setNativeInputValue(input, value) {
  const descriptor = Object.getOwnPropertyDescriptor(
    Object.getPrototypeOf(input),
    'value',
  )
  descriptor?.set?.call(input, value)
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

  const roots = getAllRoots(host)
  const inputCandidates = []

  for (const root of roots) {
    const elements = root.querySelectorAll?.('textarea, input[type="text"], input:not([type])') ?? []
    for (const element of elements) {
      if (nodeBelongsToHost(element, host)) {
        inputCandidates.push(element)
      }
    }
  }

  const targetInput = inputCandidates.at(-1)
  if (!targetInput) {
    syncStatus.value =
      'Mensaje agregado al chat espejo. No se pudo controlar el input del widget de Zapier.'
    return
  }

  setNativeInputValue(targetInput, text)
  targetInput.dispatchEvent(new Event('input', { bubbles: true }))
  targetInput.dispatchEvent(new Event('change', { bubbles: true }))

  let sendButton
  for (const root of roots) {
    const buttons = root.querySelectorAll?.('button, [role="button"]') ?? []
    for (const button of buttons) {
      if (!nodeBelongsToHost(button, host)) continue
      const label = `${button.textContent ?? ''} ${button.getAttribute?.('aria-label') ?? ''}`.toLowerCase()
      if (/send|enviar/.test(label)) {
        sendButton = button
        break
      }
    }
    if (sendButton) break
  }

  if (sendButton) {
    sendButton.click()
    syncStatus.value = 'Mensaje enviado al chat de Zapier y reflejado a la derecha.'
    return
  }

  targetInput.dispatchEvent(
    new KeyboardEvent('keydown', {
      key: 'Enter',
      code: 'Enter',
      bubbles: true,
    }),
  )

  syncStatus.value =
    'Mensaje reflejado y enviado con Enter (si el widget tiene envío por teclado habilitado).'
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
  pollTimer = setInterval(scanZapierMessages, 2000)
})

onBeforeUnmount(() => {
  observer?.disconnect()
  clearInterval(pollTimer)
  if (rafHandle) {
    cancelAnimationFrame(rafHandle)
  }
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
