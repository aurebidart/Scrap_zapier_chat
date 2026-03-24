<template>
	<div class="dashboard">
		<div class="panel">
			<div class="panel-header">
				<h1>Zapier Chat</h1>
				<p>Integración embebida</p>
			</div>
			<div class="chat-wrap">
				<zapier-interfaces-chatbot-embed ref="zapierEmbed" is-popup="false" chatbot-id="cmmdnvuvz003obp3j5uedphlx" height="600px" width="400px"></zapier-interfaces-chatbot-embed>
			</div>
		</div>

		<div class="panel">
			<div class="panel-header">
				<h1>Mock Chat</h1>
				<p>Mock diseñado — listo para interactuar</p>
			</div>
			<div class="chat-wrap">
				<MockChat :messages="messages" @send="onSend" />
			</div>
		</div>
	</div>
</template>

<script>
import { ref, onMounted } from 'vue'
import MockChat from './components/MockChat.vue'

export default {
	components: { MockChat },
	setup() {
		const messages = ref([])
		const zapierEmbed = ref(null)

		const pushMessage = (role, text) => {
			messages.value.push({ role, text })
		}

		const onSend = (text) => {
			pushMessage('user', text)
			// Optionally forward to the embed if it accepts postMessage or custom events
			try {
				console.debug('[App] outgoing user_message ->', { text })
				if (zapierEmbed.value && zapierEmbed.value.contentWindow) {
					zapierEmbed.value.contentWindow.postMessage({ type: 'user_message', text }, '*')
				}
			} catch (e) {
				console.warn('[App] failed to postMessage to embed', e)
			}
		}

		const handleZapierEvent = (e) => {
			// Generic handler: try different payload locations
			console.debug('[App] zapier event raw ->', e)
			const payload = e.detail ?? e.data ?? e
			let text = ''
			let role = 'assistant'

			if (payload && typeof payload === 'object') {
				if (payload.message && (payload.message.text || payload.message.content)) {
					text = payload.message.text ?? payload.message.content
				} else if (payload.text) {
					text = payload.text
				} else if (payload.content) {
					text = payload.content
				} else {
					text = JSON.stringify(payload)
				}
			} else {
				text = String(payload)
			}

			console.debug('[App] parsed zapier payload ->', { role, text })
			pushMessage(role, text)
		}

		onMounted(() => {
			const el = document.querySelector('zapier-interfaces-chatbot-embed')
			if (el) zapierEmbed.value = el

			// Listen to common event types and window messages
			console.info('[App] listening for window message and embed events')
			window.addEventListener('message', handleZapierEvent)
			if (el) {
				el.addEventListener('message', handleZapierEvent)
				el.addEventListener('zapier-message', handleZapierEvent)
			}
		})

		return { messages, onSend }
	}
}
</script>

