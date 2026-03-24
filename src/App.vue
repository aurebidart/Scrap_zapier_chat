<script setup>
import MockChat from './components/MockChat.vue'
</script>

<template>
  <div class="app-root">
    <div class="left">
      <h2>Zapier Chat (left)</h2>
      <div class="embed-wrap">
        <zapier-interfaces-chatbot-embed
          is-popup="false"
          chatbot-id="cmmdnvuvz003obp3j5uedphlx"
          height="600px"
          width="400px"
        ></zapier-interfaces-chatbot-embed>
      </div>
    </div>

    <div class="right">
      <MockChat />
    </div>
  </div>
</template>

<style scoped>
.app-root {
  display: flex;
  gap: 18px;
  padding: 18px;
  align-items: flex-start;
}
.left, .right {
  background: #fff;
  border: 1px solid #e6eefb;
  border-radius: 12px;
  padding: 12px;
}
.left { width: 420px }
.right { width: 420px; height: 660px; display: flex; flex-direction: column }
.embed-wrap { display:flex; justify-content:center }
</style>
