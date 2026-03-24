<template>
	<div class="dashboard">
		<div class="panel">
			<div class="panel-header">
				<h1>Zapier Chat</h1>
				<p>Integración embebida</p>
			</div>
			<div class="chat-wrap">
				<zapier-interfaces-chatbot-embed is-popup="false" chatbot-id="cmmdnvuvz003obp3j5uedphlx" height="600px" width="400px"></zapier-interfaces-chatbot-embed>
			</div>
		</div>

		<div class="panel">
			<div class="panel-header">
				<h1>Mock Chat</h1>
				<p>Mock diseñado — listo para interactuar</p>
			</div>
			<div class="chat-wrap">
				<MockChat />
			</div>
		</div>
	</div>
</template>

<script>
import MockChat from './components/MockChat.vue'

export default {
	components: { MockChat }
}
</script>

