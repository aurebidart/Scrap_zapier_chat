<template>
  <div :class="['mock-message', role]">
    <div class="meta">{{ role === 'user' ? 'Tú' : 'Bot' }}</div>
    <div class="text">{{ text }}</div>
  </div>
</template>

<script setup>
defineProps({ role: { type: String, default: 'assistant' }, text: { type: String, required: true } })
</script>

<style scoped>
.mock-message {
  max-width: 84%;
  padding: 8px 12px;
  border-radius: 12px;
  margin: 6px 0;
  display: inline-block;
}
.mock-message .meta {
  font-size: 12px;
  opacity: 0.8;
  margin-bottom: 4px;
}
.mock-message.user {
  background: #2f64ff;
  color: white;
  align-self: flex-end;
}
.mock-message.assistant {
  background: #eef2ff;
  color: #12223b;
  align-self: flex-start;
}
</style>