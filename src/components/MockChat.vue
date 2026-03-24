<template>
  <div class="mock-chat">
    <header class="mock-header">
      <h3>Chat mock (derecha)</h3>
      <p class="muted">Mocked — cada mensaje muestra "hola"</p>
    </header>

    <div class="messages" ref="box">
      <MockMessage
        v-for="(m, i) in messages"
        :key="m.id"
        :role="m.role"
        :text="m.text"
      />
    </div>

    <div class="controls">
      <button @click="addPair">Agregar par (user+bot)</button>
      <button @click="clear">Limpiar</button>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue'
import MockMessage from './MockMessage.vue'

const messages = ref([
  { id: 1, role: 'user', text: 'hola' },
  { id: 2, role: 'assistant', text: 'hola' },
])

const box = ref(null)
let counter = 3

function addPair() {
  messages.value.push({ id: counter++, role: 'user', text: 'hola' })
  messages.value.push({ id: counter++, role: 'assistant', text: 'hola' })
  nextTick(() => {
    if (box.value) box.value.scrollTop = box.value.scrollHeight
  })
}

function clear() {
  messages.value = []
}
</script>

<style scoped>
.mock-chat {
  display: flex;
  flex-direction: column;
  height: 100%;
}
.mock-header h3 { margin: 0 }
.muted { color: #6b7280; font-size: 0.9rem; margin: 4px 0 10px }
.messages {
  flex: 1;
  border: 1px solid #e5e7eb;
  border-radius: 10px;
  padding: 12px;
  overflow: auto;
  display: flex;
  flex-direction: column;
}
.controls { display:flex; gap:8px; margin-top:12px }
.controls button { padding:8px 12px; border-radius:8px; border:1px solid #c7d2fe; background:#eef2ff; cursor:pointer }
</style>