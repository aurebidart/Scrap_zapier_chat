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

			<div style="margin-top:8px; font-size:13px;">
				<strong>Embed iframe:</strong>
				<span style="margin-left:8px; color:var(--muted)">{{ iframeStatus }}</span>
				<button @click="testPostToIframe" style="margin-left:12px">Enviar prueba a iframe</button>
			</div>
		</div>

		<div class="panel">
			<div class="panel-header">
				<h1>Mock Chat</h1>
				<p>Mock diseñado — listo para interactuar</p>
			</div>
			<div class="chat-wrap">
				<MockChat :messages="messages" @send="onSend" />

				<div style="margin-top:12px">
					<strong>Eventos raw (últimos)</strong>
					<div class="raw-log">
						<div v-for="(r, i) in rawEvents" :key="i">
							<div style="opacity:.6; font-size:11px">{{ r.t }}</div>
							<pre>{{ r.v }}</pre>
						</div>
					</div>
				</div>
			</div>
		</div>
	</div>
</template>

<script>
import { ref, onMounted, onUnmounted } from 'vue'
import MockChat from './components/MockChat.vue'

export default {
	components: { MockChat },
	setup() {
		const messages = ref([])
		const zapierEmbed = ref(null)

		const pushMessage = (role, text) => {
			messages.value.push({ role, text })
		}

		const rawEvents = ref([])
		const pushRaw = (obj) => {
			rawEvents.value.unshift({ t: new Date().toISOString(), v: obj })
			if (rawEvents.value.length > 40) rawEvents.value.pop()
		}

		const zapierIframe = ref(null)
		const iframeStatus = ref('no encontrado')
		let iframePoll = null
		let mo = null

		const findIframe = () => {
			try {
				// Prefer searching inside the web component's shadowRoot
				const el = document.querySelector('zapier-interfaces-chatbot-embed')
				let iframe = null
				if (el) {
					// attempt shadowRoot
					const sr = el.shadowRoot
					if (sr) iframe = sr.querySelector('iframe')
					// fallback: search descendants
					if (!iframe) iframe = el.querySelector('iframe')
				}

				// global fallback: find a zapier iframe by src
				if (!iframe) {
					const all = [...document.querySelectorAll('iframe')]
					iframe = all.find(f => (f.src || '').includes('zapier') || (f.src || '').includes('interfaces.zapier.com'))
				}

				if (iframe) {
					zapierIframe.value = iframe
					iframeStatus.value = `encontrado — src: ${iframe.src || 'n/a'}`
					pushRaw({ type: 'info', note: 'iframe encontrado', src: iframe.src })
					return true
				}

				iframeStatus.value = 'no encontrado'
				return false
			} catch (err) {
				iframeStatus.value = 'error buscando iframe'
				return false
			}
		}

		const observeForIframe = (rootEl) => {
			try {
				if (!rootEl) return
				mo = new MutationObserver(() => {
					if (findIframe()) {
						if (mo) { mo.disconnect(); mo = null }
						if (iframePoll) { clearInterval(iframePoll); iframePoll = null }
					}
				})
				mo.observe(rootEl, { childList: true, subtree: true })
			} catch (err) {
				// ignore
			}
		}

		const testPostToIframe = () => {
			try {
				const target = zapierIframe.value || document.querySelector('zapier-interfaces-chatbot-embed')
				if (zapierIframe.value && zapierIframe.value.contentWindow) {
					zapierIframe.value.contentWindow.postMessage({ type: 'message', payload: { text: 'Prueba desde host' } }, '*')
					pushRaw({ type: 'sent-test', ok: true })
				} else if (target && target.contentWindow) {
					target.contentWindow.postMessage({ type: 'message', payload: { text: 'Prueba desde host' } }, '*')
					pushRaw({ type: 'sent-test', ok: true })
				} else {
					pushRaw({ type: 'sent-test', ok: false, reason: 'no iframe/contentWindow' })
				}
			} catch (err) {
				pushRaw({ type: 'sent-test', ok: false, err: String(err) })
			}
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
			pushRaw({ type: 'raw', e: e })

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
			pushRaw({ type: 'parsed', role, text })
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
				// Observe for dynamically added iframe inside the component
				observeForIframe(el)
			}

			// Start polling for an iframe for a short period in case it's created asynchronously
			iframePoll = setInterval(() => {
				if (findIframe()) {
					clearInterval(iframePoll)
					iframePoll = null
				}
			}, 800)
			// stop polling after 12s
			setTimeout(() => { if (iframePoll) { clearInterval(iframePoll); iframePoll = null } }, 12000)
		})

		onUnmounted(() => {
			window.removeEventListener('message', handleZapierEvent)
			const el = document.querySelector('zapier-interfaces-chatbot-embed')
			if (el) {
				el.removeEventListener('message', handleZapierEvent)
				el.removeEventListener('zapier-message', handleZapierEvent)
			}
			if (mo) { mo.disconnect(); mo = null }
			if (iframePoll) { clearInterval(iframePoll); iframePoll = null }
		})

		return { messages, onSend, rawEvents, iframeStatus, testPostToIframe }
	}
}
</script>

<style>
.raw-log {
	max-height: 220px;
	overflow: auto;
	background: #0f1724;
	color: #d1d5db;
	padding: 8px;
	font-size: 12px;
	border-radius: 6px;
}
.raw-log pre { margin: 4px 0; white-space: pre-wrap; word-break: break-word }
</style>

