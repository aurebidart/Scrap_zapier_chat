<template>
  <div>
    <div class="mirror-thread" ref="thread">
      <div v-for="(m, i) in displayMessages" :key="i" :class="['bubble', m.role === 'user' ? 'user' : 'assistant']">
        <strong v-if="m.role === 'assistant'">Assistant</strong>
        <strong v-else>Tú</strong>
        {{ m.text }}
      </div>
      <div v-if="!displayMessages || displayMessages.length === 0" class="empty-state">No hay mensajes aún. Escribe abajo para comenzar.</div>
    </div>

    <div class="input-bar" style="margin-top:8px">
      <input v-model="input" @keyup.enter="send" placeholder="Escribe un mensaje..." />
      <button @click="send">Enviar</button>
    </div>
  </div>
</template>

<script>
import { ref, nextTick, computed, watch } from 'vue'

export default {
  props: {
    messages: { type: Array, default: null }
  },
  emits: ['send'],
  setup(props, { emit }) {
    const internalMessages = ref([
      { role: 'assistant', text: 'Hola — soy un chat mock. Puedo simular respuestas rápidas.' },
      { role: 'user', text: 'Perfecto, probemos.' },
      { role: 'assistant', text: 'Genial. ¿Qué tema quieres simular?' }
    ])

    const input = ref('')
    const thread = ref(null)

    const displayMessages = computed(() => {
      return props.messages && props.messages.length ? props.messages : internalMessages.value
    })

    const scrollToBottom = async () => {
      await nextTick()
      if (thread.value) thread.value.scrollTop = thread.value.scrollHeight
    }

    watch(displayMessages, () => scrollToBottom(), { deep: true })

    const send = () => {
      if (!input.value.trim()) return
      const text = input.value.trim()
      // Emit to parent so the parent can decide what to do with the outgoing message
      emit('send', text)

      // Keep local copy so UI shows immediate feedback if we're using internal messages
      if (!props.messages) {
        internalMessages.value.push({ role: 'user', text })
      }

      input.value = ''
      scrollToBottom()
    }

    return { displayMessages, input, send, thread }
  }
}
</script>
