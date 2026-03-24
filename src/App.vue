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
			console.debug('[App] zapier event raw ->', e)

			// Normalize incoming payload from possible sources (CustomEvent.detail, postMessage.data, or direct event)
			const raw = e?.detail ?? e?.data ?? e
			let payload = raw

			// If payload is a string, try to parse JSON. If not JSON, keep as string.
			if (typeof raw === 'string') {
				try {
					payload = JSON.parse(raw)
				} catch (err) {
					payload = raw
				}
			}

			// Handle simple string control messages from the embed
			if (typeof payload === 'string') {
				if (payload === 'zChatbotReady') {
					console.info('[App] zapier embed ready event received')
					pushMessage('assistant', 'Zapier embed ready')
					return
				}
				pushMessage('assistant', payload)
				return
			}

			// For object payloads, try common shapes used by embeds
			let text = ''
			let role = 'assistant'

			try {
				if (payload?.type === 'message' && payload?.payload) {
					const msg = payload.payload
					text = msg?.text ?? msg?.content ?? msg?.body ?? ''
					role = msg?.role ?? msg?.sender ?? role
				} else if (payload?.message) {
					const msg = payload.message
					text = msg?.text ?? msg?.content ?? msg?.body ?? ''
					role = msg?.role ?? msg?.sender ?? role
				} else if (payload?.data?.message) {
					const msg = payload.data.message
					text = msg?.text ?? msg?.content ?? ''
					role = msg?.role ?? role
				} else if (payload?.text || payload?.content) {
					text = payload.text ?? payload.content
					role = payload.role ?? role
				} else {
					text = JSON.stringify(payload)
				}
			} catch (err) {
				text = JSON.stringify(payload)
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

