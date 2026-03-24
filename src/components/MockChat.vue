<template>
  <div>
    <div class="mirror-thread" ref="thread">
      <div v-for="(m, i) in messages" :key="i" :class="['bubble', m.role === 'user' ? 'user' : 'assistant']">
        <strong v-if="m.role === 'assistant'">Assistant</strong>
        <strong v-else>Tú</strong>
        {{ m.text }}
      </div>
      <div v-if="messages.length === 0" class="empty-state">No hay mensajes aún. Escribe abajo para comenzar.</div>
    </div>

    <div class="input-bar" style="margin-top:8px">
      <input v-model="input" @keyup.enter="send" placeholder="Escribe un mensaje..." />
      <button @click="send">Enviar</button>
    </div>
  </div>
</template>

<script>
import { ref, nextTick } from 'vue'

export default {
  setup() {
    const messages = ref([
      { role: 'assistant', text: 'Hola — soy un chat mock. Puedo simular respuestas rápidas.' },
      { role: 'user', text: 'Perfecto, probemos.' },
      { role: 'assistant', text: 'Genial. ¿Qué tema quieres simular?' }
    ])

    const input = ref('')
    const thread = ref(null)

    const scrollToBottom = async () => {
      await nextTick()
      if (thread.value) thread.value.scrollTop = thread.value.scrollHeight
    }

    const send = () => {
      if (!input.value.trim()) return
      const text = input.value.trim()
      messages.value.push({ role: 'user', text })
      input.value = ''
      scrollToBottom()

      // Simulate assistant reply
      setTimeout(() => {
        messages.value.push({ role: 'assistant', text: `Mock respuesta a: "${text}" — (generado)` })
        scrollToBottom()
      }, 700)
    }

    return { messages, input, send, thread }
  }
}
</script>
