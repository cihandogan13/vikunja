<template>
	<XButton
		v-if="!isPlanRoute"
		class="capture-fab"
		variant="primary"
		icon="microphone"
		:aria-label="$t('capture.open')"
		@click="openPanel"
	>
		{{ $t('capture.open') }}
	</XButton>

	<Modal
		:enabled="open"
		variant="sheet"
		:title="$t('capture.title')"
		@close="closePanel"
	>
		<template #header-action>
			<BaseButton
				:aria-label="$t('capture.close')"
				@click="closePanel"
			>
				<Icon icon="times" />
			</BaseButton>
		</template>

		<div class="capture-panel">
			<template v-if="!draft">
				<div class="capture-panel__modes">
					<XButton
						:variant="mode === 'text' ? 'primary' : 'tertiary'"
						:shadow="false"
						@click="setMode('text')"
					>
						{{ $t('capture.mode.text') }}
					</XButton>
					<XButton
						:variant="mode === 'voice' ? 'primary' : 'tertiary'"
						:shadow="false"
						@click="setMode('voice')"
					>
						{{ $t('capture.mode.voice') }}
					</XButton>
				</div>

				<template v-if="mode === 'text'">
					<p class="capture-panel__hint">
						{{ $t('capture.placeholder') }}
					</p>
					<textarea
						v-model="text"
						class="input"
						rows="4"
						:disabled="loading"
						:placeholder="$t('capture.placeholder')"
					/>
					<div class="capture-panel__actions">
						<XButton
							variant="primary"
							:shadow="false"
							:loading="loading"
							:disabled="loading"
							@click="submit"
						>
							{{ loading ? $t('capture.sending') : $t('capture.send') }}
						</XButton>
					</div>
				</template>

				<template v-else>
					<p class="capture-panel__hint">
						{{ $t('capture.voiceHint') }}
					</p>
					<div
						v-if="recording"
						class="capture-panel__recording"
					>
						<span class="capture-panel__dot" />
						<span>{{ $t('capture.recording', {time: recordingTime}) }}</span>
						<XButton
							variant="primary"
							:shadow="false"
							@click="stopRecording"
						>
							{{ $t('capture.stopRecording') }}
						</XButton>
					</div>
					<div
						v-else
						class="capture-panel__actions"
					>
						<XButton
							variant="primary"
							icon="microphone"
							:shadow="false"
							:loading="loading"
							:disabled="loading"
							@click="startRecording"
						>
							{{ loading ? $t('capture.processingVoice') : $t('capture.startRecording') }}
						</XButton>
					</div>
				</template>

				<p class="capture-panel__commands">
					{{ $t('capture.commands') }}
				</p>

				<p
					v-if="errorMessage"
					class="has-text-danger"
					role="alert"
				>
					{{ errorMessage }}
				</p>
			</template>

			<template v-else>
				<p class="capture-panel__hint">
					{{ $t('capture.draftHint') }}
				</p>

				<p
					v-if="transcript"
					class="capture-panel__transcript"
				>
					<strong>{{ $t('capture.transcript') }}:</strong> {{ transcript }}
				</p>

				<label class="label">{{ $t('capture.fields.title') }}</label>
				<input
					v-model="draft.title"
					class="input"
				>

				<label class="label">{{ $t('capture.fields.description') }}</label>
				<textarea
					v-model="draft.description"
					class="input"
					rows="3"
				/>

				<div class="capture-panel__grid">
					<div>
						<label class="label">{{ $t('capture.fields.project') }}</label>
						<select
							v-model.number="draft.project_id"
							class="input"
						>
							<option
								v-for="project in projects"
								:key="project.id"
								:value="project.id"
							>
								{{ project.title }}
							</option>
						</select>
					</div>
					<div>
						<label class="label">{{ $t('capture.fields.duration') }}</label>
						<input
							v-model.number="draft.duration_min"
							class="input"
							type="number"
							min="5"
							max="480"
						>
					</div>
					<div>
						<label class="label">{{ $t('capture.fields.dueDate') }}</label>
						<input
							v-model="dueDateLocal"
							class="input"
							type="datetime-local"
						>
					</div>
					<div>
						<label class="label">{{ $t('capture.fields.importance') }}</label>
						<select
							v-model="draft.importance"
							class="input"
						>
							<option value="low">{{ $t('capture.level.low') }}</option>
							<option value="medium">{{ $t('capture.level.medium') }}</option>
							<option value="high">{{ $t('capture.level.high') }}</option>
						</select>
					</div>
					<div>
						<label class="label">{{ $t('capture.fields.urgency') }}</label>
						<select
							v-model="draft.urgency"
							class="input"
						>
							<option value="low">{{ $t('capture.level.low') }}</option>
							<option value="medium">{{ $t('capture.level.medium') }}</option>
							<option value="high">{{ $t('capture.level.high') }}</option>
						</select>
					</div>
				</div>

				<div class="capture-panel__divider" />

				<label class="label">{{ $t('capture.fields.labels') }}</label>
				<input
					v-model="labelsText"
					class="input capture-panel__labels"
					:placeholder="$t('capture.labelsPlaceholder')"
				>

				<div class="capture-panel__grid">
					<div>
						<label class="label">{{ $t('capture.fields.repeat') }}</label>
						<div class="capture-panel__repeat">
							<input
								v-model.number="draft.repeat_every"
								class="input"
								type="number"
								min="1"
								max="3650"
								:placeholder="$t('capture.repeatEvery')"
							>
							<select
								v-model="draft.repeat_unit"
								class="input"
							>
								<option value="">{{ $t('capture.repeatUnits.none') }}</option>
								<option value="hour">{{ $t('capture.repeatUnits.hour') }}</option>
								<option value="day">{{ $t('capture.repeatUnits.day') }}</option>
								<option value="week">{{ $t('capture.repeatUnits.week') }}</option>
								<option value="month">{{ $t('capture.repeatUnits.month') }}</option>
								<option value="year">{{ $t('capture.repeatUnits.year') }}</option>
							</select>
						</div>
					</div>
					<div>
						<label class="label">{{ $t('capture.fields.priority') }}</label>
						<select
							v-model="draft.priority"
							class="input capture-panel__priority"
						>
							<option :value="null">{{ $t('capture.priorityAuto') }}</option>
							<option
								v-for="level in 5"
								:key="level"
								:value="level"
							>
								{{ level }}
							</option>
						</select>
					</div>
					<div>
						<label class="label">{{ $t('capture.fields.reminder') }}</label>
						<input
							v-model.number="draft.reminder_minutes"
							class="input capture-panel__reminder"
							type="number"
							min="1"
							:placeholder="$t('capture.reminderPlaceholder')"
						>
					</div>
				</div>

				<div class="capture-panel__subtasks">
					<div class="capture-panel__subtasks-header">
						<span class="label">{{ $t('capture.subtasks.title') }}</span>
						<XButton
							variant="tertiary"
							icon="plus"
							:shadow="false"
							@click="addSubtask"
						>
							{{ $t('capture.subtasks.add') }}
						</XButton>
					</div>
					<div
						v-for="(subtask, index) in draft.subtasks"
						:key="index"
						class="capture-panel__subtask"
					>
						<input
							v-model="subtask.title"
							class="input"
							:placeholder="$t('capture.subtasks.placeholder')"
						>
						<input
							v-model.number="subtask.duration_min"
							class="input capture-panel__subtask-duration"
							type="number"
							min="5"
							max="240"
							:placeholder="$t('capture.subtasks.duration')"
						>
						<BaseButton
							:aria-label="$t('capture.subtasks.remove')"
							@click="removeSubtask(index)"
						>
							<Icon icon="times" />
						</BaseButton>
					</div>
				</div>

				<p
					v-if="errorMessage"
					class="has-text-danger"
					role="alert"
				>
					{{ errorMessage }}
				</p>

				<p
					v-if="committedTaskId !== null"
					class="capture-panel__success"
					role="status"
				>
					{{ $t('capture.committed', {id: committedTaskId}) }}
				</p>

				<div class="capture-panel__actions">
					<XButton
						variant="tertiary"
						:shadow="false"
						@click="reset"
					>
						{{ $t('capture.new') }}
					</XButton>
					<XButton
						v-if="committedTaskId !== null"
						variant="primary"
						:shadow="false"
						@click="openTask"
					>
						{{ $t('capture.openTask') }}
					</XButton>
					<XButton
						v-else
						variant="primary"
						:shadow="false"
						:loading="commitLoading"
						:disabled="commitLoading"
						@click="commitDraft"
					>
						{{ commitLoading ? $t('capture.committing') : $t('capture.commit') }}
					</XButton>
				</div>
			</template>
		</div>
	</Modal>
</template>

<script lang="ts" setup>
import {computed, onBeforeUnmount, ref} from 'vue'
import {useI18n} from 'vue-i18n'
import {useRoute, useRouter} from 'vue-router'

import BaseButton from '@/components/base/BaseButton.vue'
import {getToken} from '@/helpers/auth'

type Level = 'low' | 'medium' | 'high'

type SubtaskDraft = {
	title: string,
	duration_min: number | null,
}

type CaptureDraft = {
	title: string,
	description: string,
	project_id: number,
	due_date: string | null,
	importance: Level,
	urgency: Level,
	duration_min: number,
	labels: string[],
	repeat_every: number | null,
	repeat_unit: '' | 'hour' | 'day' | 'week' | 'month' | 'year',
	priority: number | null,
	reminder_minutes: number | null,
	split_requested: boolean,
	subtasks: SubtaskDraft[],
	confidence: number,
}

type CaptureResponse = {
	capture_id: string,
	source: 'text' | 'voice',
	transcript: string,
	draft: CaptureDraft,
}

type ProjectOption = {
	id: number,
	title: string,
}

type CommitResponse = {
	capture_id: string,
	task_id: number,
	subtask_ids: number[],
}

const {t} = useI18n({useScope: 'global'})
const route = useRoute()
const router = useRouter()
const isPlanRoute = computed(() => route.name === 'plan')
const open = ref(false)
const mode = ref<'text' | 'voice'>('text')
const text = ref('')
const transcript = ref('')
const loading = ref(false)
const commitLoading = ref(false)
const errorMessage = ref('')
const draft = ref<CaptureDraft | null>(null)
const projects = ref<ProjectOption[]>([])
const committedTaskId = ref<number | null>(null)
const recording = ref(false)
const recordingSeconds = ref(0)
const recordingTime = computed(() => {
	const minutes = String(Math.floor(recordingSeconds.value / 60)).padStart(2, '0')
	const seconds = String(recordingSeconds.value % 60).padStart(2, '0')
	return `${minutes}:${seconds}`
})
const dueDateLocal = computed({
	get(): string {
		const value = draft.value?.due_date
		if (!value) {
			return ''
		}
		return value.length === 10 ? `${value}T23:59` : value.slice(0, 16)
	},
	set(value: string): void {
		if (!draft.value) {
			return
		}
		if (!value) {
			draft.value.due_date = null
			return
		}
		const normalized = value.length === 16 ? `${value}:00` : value
		draft.value.due_date = `${normalized}+03:00`
	},
})

const labelsText = computed({
	get(): string {
		return draft.value?.labels?.join(', ') ?? ''
	},
	set(value: string): void {
		if (!draft.value) {
			return
		}
		draft.value.labels = value
			.split(',')
			.map(label => label.trim().replace(/^#/, ''))
			.filter(Boolean)
	},
})
let projectsLoaded = false
let captureId = ''
let recorder: MediaRecorder | null = null
let mediaStream: MediaStream | null = null
let chunks: BlobPart[] = []
let recordingTimer: number | undefined
let recordingMaxTimer: number | undefined
let discardRecording = false

function newCaptureId(): string {
	if (typeof crypto.randomUUID === 'function') {
		return crypto.randomUUID()
	}
	return `${Date.now()}-${Math.random().toString(16).slice(2)}`
}

function openPanel(): void {
	if (!captureId) {
		captureId = newCaptureId()
	}
	open.value = true
	if (!projectsLoaded) {
		void loadProjects()
	}
}

async function loadProjects(): Promise<void> {
	const token = getToken()
	if (!token) {
		errorMessage.value = 'Oturum bulunamadı. Lütfen tekrar giriş yap.'
		return
	}

	try {
		const response = await fetch('/capture/api/projects', {
			headers: {Authorization: `Bearer ${token}`},
		})
		const body = await response.json().catch(() => ({}))
		if (!response.ok) {
			throw new Error(body?.error?.message ?? `HTTP ${response.status}`)
		}
		projects.value = body.items ?? []
		projectsLoaded = true
	} catch (e) {
		errorMessage.value = e instanceof Error ? e.message : 'Projeler yüklenemedi.'
	}
}

function closePanel(): void {
	open.value = false
	reset()
}

function setMode(newMode: 'text' | 'voice'): void {
	if (recording.value) {
		cancelRecording()
	}
	mode.value = newMode
	errorMessage.value = ''
}

function clearRecordingTimers(): void {
	if (recordingTimer !== undefined) {
		window.clearInterval(recordingTimer)
		recordingTimer = undefined
	}
	if (recordingMaxTimer !== undefined) {
		window.clearTimeout(recordingMaxTimer)
		recordingMaxTimer = undefined
	}
}

function stopMediaStream(): void {
	mediaStream?.getTracks().forEach(track => track.stop())
	mediaStream = null
}

function cancelRecording(): void {
	discardRecording = true
	clearRecordingTimers()
	if (recorder && recorder.state !== 'inactive') {
		try {
			recorder.stop()
		} catch {
			// Recorder already stopped.
		}
	}
	recording.value = false
	recordingSeconds.value = 0
	stopMediaStream()
}

async function startRecording(): Promise<void> {
	if (recording.value || loading.value) {
		return
	}
	errorMessage.value = ''

	if (!navigator.mediaDevices?.getUserMedia || typeof MediaRecorder === 'undefined') {
		errorMessage.value = t('capture.micUnsupported')
		return
	}

	try {
		const activeStream = await navigator.mediaDevices.getUserMedia({audio: true})
		const candidates = ['audio/webm;codecs=opus', 'audio/webm', 'audio/mp4']
		const mimeType = typeof MediaRecorder.isTypeSupported === 'function'
			? candidates.find(candidate => MediaRecorder.isTypeSupported(candidate)) ?? ''
			: ''
		const activeRecorder = mimeType
			? new MediaRecorder(activeStream, {mimeType})
			: new MediaRecorder(activeStream)

		mediaStream = activeStream
		recorder = activeRecorder
		chunks = []
		discardRecording = false

		activeRecorder.ondataavailable = event => {
			if (event.data.size > 0) {
				chunks.push(event.data)
			}
		}
		activeRecorder.onstop = () => {
			const shouldSend = !discardRecording
			discardRecording = false
			clearRecordingTimers()
			recording.value = false
			recordingSeconds.value = 0
			stopMediaStream()
			const type = activeRecorder.mimeType || mimeType || 'audio/webm'
			const blob = new Blob(chunks, {type})
			chunks = []
			if (shouldSend) {
				void sendVoice(blob)
			}
		}

		activeRecorder.start()
		recording.value = true
		recordingSeconds.value = 0
		recordingTimer = window.setInterval(() => {
			recordingSeconds.value += 1
		}, 1000)
		recordingMaxTimer = window.setTimeout(() => {
			stopRecording()
		}, 120000)
	} catch (error) {
		stopMediaStream()
		recording.value = false
		if (error instanceof DOMException && error.name === 'NotAllowedError') {
			errorMessage.value = t('capture.micDenied')
		} else {
			errorMessage.value = error instanceof Error ? error.message : t('capture.micUnsupported')
		}
	}
}

function stopRecording(): void {
	clearRecordingTimers()
	recording.value = false
	if (recorder && recorder.state !== 'inactive') {
		recorder.stop()
	}
}

async function sendVoice(blob: Blob): Promise<void> {
	const token = getToken()
	if (!token) {
		errorMessage.value = t('capture.sessionMissing')
		return
	}

	errorMessage.value = ''
	loading.value = true
	try {
		const extension = blob.type.includes('mp4') ? 'mp4' : 'webm'
		const form = new FormData()
		form.set('capture_id', captureId)
		form.set('source', 'voice')
		form.set('audio', blob, `capture.${extension}`)

		const response = await fetch('/capture/api/captures', {
			method: 'POST',
			headers: {Authorization: `Bearer ${token}`},
			body: form,
		})
		const body = await response.json().catch(() => ({}))
		if (!response.ok) {
			throw new Error(body?.error?.message ?? `HTTP ${response.status}`)
		}

		const result = body as CaptureResponse
		transcript.value = result.transcript
		draft.value = result.draft
	} catch (error) {
		errorMessage.value = error instanceof Error ? error.message : t('capture.audioFailed')
	} finally {
		loading.value = false
	}
}

function reset(): void {
	cancelRecording()
	mode.value = 'text'
	text.value = ''
	transcript.value = ''
	draft.value = null
	errorMessage.value = ''
	loading.value = false
	commitLoading.value = false
	committedTaskId.value = null
	captureId = ''
}

onBeforeUnmount(() => {
	cancelRecording()
})

function addSubtask(): void {
	if (draft.value && draft.value.subtasks.length < 10) {
		draft.value.subtasks.push({title: '', duration_min: null})
	}
}

function removeSubtask(index: number): void {
	draft.value?.subtasks.splice(index, 1)
}

async function commitDraft(): Promise<void> {
	if (!draft.value || !captureId) {
		return
	}

	const token = getToken()
	if (!token) {
		errorMessage.value = 'Oturum bulunamadı. Lütfen tekrar giriş yap.'
		return
	}

	errorMessage.value = ''
	commitLoading.value = true
	try {
		const response = await fetch(`/capture/api/captures/${captureId}/commit`, {
			method: 'POST',
			headers: {
				Authorization: `Bearer ${token}`,
				'Content-Type': 'application/json',
			},
			body: JSON.stringify({
				...draft.value,
				labels: draft.value.labels,
				repeat_every: draft.value.repeat_every && Number(draft.value.repeat_every) > 0
					? Number(draft.value.repeat_every)
					: null,
				repeat_unit: draft.value.repeat_unit || '',
				reminder_minutes: draft.value.reminder_minutes && Number(draft.value.reminder_minutes) > 0
					? Number(draft.value.reminder_minutes)
					: null,
				subtasks: draft.value.subtasks
					.map(subtask => ({
						title: subtask.title.trim(),
						duration_min: subtask.duration_min === null
							|| subtask.duration_min === undefined
							|| Number(subtask.duration_min) <= 0
							? null
							: Number(subtask.duration_min),
					}))
					.filter(subtask => subtask.title.length > 0),
			}),
		})
		const body = await response.json().catch(() => ({}))
		if (!response.ok) {
			throw new Error(body?.error?.message ?? `HTTP ${response.status}`)
		}
		committedTaskId.value = (body as CommitResponse).task_id
	} catch (e) {
		errorMessage.value = e instanceof Error ? e.message : 'Görev oluşturulamadı.'
	} finally {
		commitLoading.value = false
	}
}

function openTask(): void {
	if (committedTaskId.value === null) {
		return
	}
	router.push({name: 'task.detail', params: {id: committedTaskId.value}})
	closePanel()
}

async function submit(): Promise<void> {
	errorMessage.value = ''

	const value = text.value.trim()
	if (value === '') {
		errorMessage.value = 'Bir görev yazmalısın.'
		return
	}

	loading.value = true
	try {
		const form = new FormData()
		form.set('capture_id', captureId)
		form.set('source', 'text')
		form.set('text', value)

		const token = getToken()
		const headers: Record<string, string> = {}
		if (token) {
			headers.Authorization = `Bearer ${token}`
		}

		const response = await fetch('/capture/api/captures', {
			method: 'POST',
			headers,
			body: form,
		})
		const body = await response.json().catch(() => ({}))
		if (!response.ok) {
			throw new Error(body?.error?.message ?? `HTTP ${response.status}`)
		}

		const result = body as CaptureResponse
		transcript.value = result.transcript
		draft.value = result.draft
	} catch (e) {
		errorMessage.value = e instanceof Error ? e.message : 'Yakalama başarısız oldu.'
	} finally {
		loading.value = false
	}
}
</script>

<style lang="scss" scoped>
.capture-fab {
	position: fixed;
	inset-block-end: 4.5rem;
	inset-inline-end: 1rem;
	z-index: 3000;
	padding: 0.75rem 1.1rem;
	border-radius: 999px;
	box-shadow: var(--shadow-lg, 0 6px 20px rgb(0 0 0 / 25%));
	text-transform: none;
	font-size: 0.95rem;

	@media screen and (max-width: $tablet) {
		inset-block-end: 1rem;
		padding: 0.9rem;
		border-radius: 50%;

		> span:last-child {
			display: none;
		}
	}
}

.capture-panel {
	display: flex;
	flex-direction: column;
	gap: 0.6rem;
	padding: 0.5rem 1rem 1rem;
	inline-size: min(32rem, 86vw);
	margin-inline: auto;
	max-block-size: min(78vh, 48rem);
	overflow-y: auto;
	overscroll-behavior: contain;

	&__hint {
		color: var(--grey-500);
		font-size: 0.9rem;
		margin-block-end: 0.25rem;
	}

	&__modes {
		display: flex;
		gap: 0.5rem;
		margin-block-end: 0.25rem;
	}

	&__recording {
		display: flex;
		align-items: center;
		gap: 0.6rem;
		color: var(--danger, #f14668);
	}

	&__dot {
		inline-size: 0.7rem;
		block-size: 0.7rem;
		border-radius: 50%;
		background: var(--danger, #f14668);
		animation: capture-pulse 1s infinite;
	}

	&__transcript {
		background: var(--grey-100);
		padding: 0.5rem 0.75rem;
		border-radius: $radius;
		font-size: 0.9rem;
		overflow-wrap: anywhere;
	}

	&__grid {
		display: grid;
		grid-template-columns: repeat(2, minmax(0, 1fr));
		gap: 0.6rem;

		@media screen and (max-width: $tablet) {
			grid-template-columns: 1fr;
		}
	}

	&__divider {
		border-block-start: 1px solid var(--grey-200);
		margin-block: 0.35rem;
	}

	&__repeat {
		display: flex;
		gap: 0.4rem;

		input {
			inline-size: 5.5rem;
			flex: 0 0 auto;
		}
	}

	&__commands {
		color: var(--grey-500);
		font-size: 0.8rem;
		line-height: 1.4;
		margin-block-start: 0.25rem;
	}

	&__subtasks {
		display: flex;
		flex-direction: column;
		gap: 0.4rem;
	}

	&__subtasks-header {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 0.5rem;

		.label {
			margin-block-end: 0;
		}
	}

	&__subtask {
		display: grid;
		grid-template-columns: minmax(0, 1fr) 5.5rem auto;
		gap: 0.4rem;
		align-items: center;
	}

	&__subtask-duration {
		text-align: center;
	}

	&__success {
		color: var(--success);
		font-weight: 600;
		margin-block-start: 0.5rem;
	}

	&__actions {
		display: flex;
		justify-content: flex-end;
		gap: 0.5rem;
		margin-block-start: 0.5rem;
	}
}
@keyframes capture-pulse {
	0%, 100% { opacity: 1; }
	50% { opacity: 0.35; }
}
</style>
