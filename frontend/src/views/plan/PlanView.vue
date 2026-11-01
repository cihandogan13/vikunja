<template>
	<div class="content plan-view">
		<header class="plan-head">
			<div class="plan-head__intro">
				<p class="plan-head__eyebrow">
					{{ $t('plan.eyebrow') }}
				</p>
				<h1 class="plan-head__date">
					<span class="plan-head__day-number">{{ dayParts.day }}</span>
					<span class="plan-head__date-text">
						<span class="plan-head__month">{{ dayParts.monthYear }}</span>
						<span class="plan-head__weekday">{{ dayParts.weekday }}</span>
					</span>
				</h1>
				<p class="plan-head__window">
					{{ planMeta }}
				</p>
			</div>
			<div class="plan-head__side">
				<div
					class="plan-head__live"
					:class="{'is-busy': running || refreshing}"
					aria-live="polite"
				>
					<span
						class="plan-head__live-dot"
						aria-hidden="true"
					/>
					<span>{{ running ? $t('plan.updating') : $t('plan.live') }}</span>
					<span
						v-if="!running && lastSync"
						class="plan-head__live-time"
					>{{ $t('plan.updated') }} {{ lastSync }}</span>
				</div>
				<div class="plan-head__controls">
					<button
						class="icon-button"
						type="button"
						:aria-label="$t('plan.previousDay')"
						@click="shiftDate(-1)"
					>
						<svg
							viewBox="0 0 24 24"
							aria-hidden="true"
						>
							<path d="m15 18-6-6 6-6" />
						</svg>
					</button>
					<button
						class="today-button"
						type="button"
						:disabled="isToday"
						@click="goToday"
					>
						{{ $t('plan.today') }}
					</button>
					<button
						class="icon-button"
						type="button"
						:aria-label="$t('plan.nextDay')"
						@click="shiftDate(1)"
					>
						<svg
							viewBox="0 0 24 24"
							aria-hidden="true"
						>
							<path d="m9 18 6-6-6-6" />
						</svg>
					</button>
					<label class="date-picker">
						<svg
							viewBox="0 0 24 24"
							aria-hidden="true"
						>
							<rect
								x="3"
								y="5"
								width="18"
								height="16"
								rx="2"
							/>
							<path d="M16 3v4M8 3v4M3 11h18" />
						</svg>
						<span class="visually-hidden">{{ $t('plan.date') }}</span>
						<input
							v-model="selectedDate"
							type="date"
							:aria-label="$t('plan.date')"
							@change="onDateChange"
						>
					</label>
				</div>
			</div>
		</header>

		<nav
			class="week-strip"
			:aria-label="$t('plan.week')"
		>
			<p class="week-strip__range">
				{{ weekRange }}
			</p>
			<div class="week-strip__days">
				<button
					v-for="day in weekDays"
					:key="day.iso"
					class="week-strip__day"
					:class="{
						'is-selected': day.isSelected,
						'is-today': day.isToday,
						'is-weekend': day.isWeekend,
					}"
					type="button"
					:aria-pressed="day.isSelected"
					@click="selectDate(day.iso)"
				>
					<span class="week-strip__weekday">{{ day.weekday }}</span>
					<span class="week-strip__number">{{ day.day }}</span>
					<span
						v-if="day.isToday"
						class="week-strip__today"
						aria-hidden="true"
					/>
				</button>
			</div>
		</nav>

		<p
			v-if="errorMessage"
			class="plan-alert"
			role="alert"
		>
			{{ errorMessage }}
		</p>

		<div
			v-if="loading && !plan"
			class="plan-loading"
			aria-live="polite"
		>
			<span
				class="plan-loading__spinner"
				aria-hidden="true"
			/>
			{{ $t('plan.loading') }}
		</div>

		<template v-else-if="plan">
			<div class="plan-grid">
				<section
					class="plan-calendar"
					:aria-label="$t('plan.scheduled')"
					:aria-busy="running || refreshing"
				>
					<div
						class="calendar"
						:style="calendarStyle"
					>
						<div
							class="calendar__gutter"
							aria-hidden="true"
						>
							<span
								v-for="hour in hours"
								:key="hour"
								class="calendar__hour"
							>{{ hour }}</span>
						</div>
						<div class="calendar__canvas">
							<div
								class="calendar__grid"
								aria-hidden="true"
							/>
							<div
								v-if="nowTop !== null"
								class="calendar__now"
								:style="{top: `${nowTop}px`}"
								aria-hidden="true"
							>
								<span class="calendar__now-dot" />
								<span class="calendar__now-label">{{ $t('plan.now') }}</span>
							</div>
							<RouterLink
								v-for="block in laidOutBlocks"
								:key="`${block.task_id}-${block.start}`"
								:to="{name: 'task.detail', params: {id: block.task_id}}"
								class="event"
								:class="eventClasses(block)"
								:style="eventStyle(block)"
								:aria-label="eventAria(block)"
							>
								<span class="event__time">{{ time(block.start) }}–{{ time(block.end) }}</span>
								<span class="event__title">{{ block.title }}</span>
								<span class="event__meta">
									<span>{{ $t('plan.minutes', {count: block.duration_min}) }}</span>
									<span class="event__source">{{ block.source === 'manual' ? $t('plan.manual') : $t('plan.planned') }}</span>
									<span
										class="event__importance"
										:class="`is-${block.importance}`"
									>
										<span
											class="event__dot"
											aria-hidden="true"
										/>
										{{ $t(`plan.importance.${block.importance}`) }}
									</span>
								</span>
							</RouterLink>
							<div
								v-if="plan.scheduled.length === 0"
								class="calendar__empty"
							>
								<p class="calendar__empty-title">
									{{ $t('plan.emptyCalendar') }}
								</p>
								<p class="calendar__empty-hint">
									{{ $t('plan.emptyHint') }}
								</p>
							</div>
						</div>
					</div>
					<div class="calendar__legend">
						<span><span class="legend-swatch is-manual" />{{ $t('plan.manual') }}</span>
						<span><span class="legend-swatch is-planned" />{{ $t('plan.planned') }}</span>
						<span><span class="legend-swatch is-high" />{{ $t('plan.importance.high') }}</span>
					</div>
				</section>

				<aside class="plan-rail">
					<section class="rail-block rail-block--stats">
						<p class="rail-label">
							{{ $t('plan.summary') }}
						</p>
						<div class="rail-stats">
							<div>
								<strong>{{ $t('plan.taskCount', {count: plan.scheduled.length}) }}</strong>
								<span>{{ $t('plan.plannedTotal', {count: scheduledMinutes}) }}</span>
							</div>
							<div>
								<strong>{{ $t('plan.freeShort', {count: freeMinutes}) }}</strong>
								<span>{{ $t('plan.windowShort', {start: windowLabel.start, end: windowLabel.end}) }}</span>
							</div>
						</div>
					</section>

					<section class="rail-block">
						<p class="rail-label">
							{{ $t('plan.nextUp') }}
						</p>
						<RouterLink
							v-if="nextBlock"
							:to="{name: 'task.detail', params: {id: nextBlock.task_id}}"
							class="next-card"
						>
							<span class="next-card__time">{{ time(nextBlock.start) }}–{{ time(nextBlock.end) }}</span>
							<span class="next-card__title">{{ nextBlock.title }}</span>
							<span class="next-card__meta">
								{{ $t('plan.minutes', {count: nextBlock.duration_min}) }} ·
								{{ nextBlock.source === 'manual' ? $t('plan.manual') : $t('plan.planned') }}
							</span>
						</RouterLink>
						<p
							v-else
							class="rail-empty"
						>
							{{ $t('plan.noNext') }}
						</p>
					</section>

					<section class="rail-block">
						<p class="rail-label">
							{{ $t('plan.unscheduled') }}
						</p>
						<div
							v-if="plan.unscheduled.length"
							class="rail-list"
						>
							<RouterLink
								v-for="task in plan.unscheduled"
								:key="task.task_id"
								:to="{name: 'task.detail', params: {id: task.task_id}}"
								class="rail-item"
							>
								<span class="rail-item__title">{{ task.title }}</span>
								<span class="rail-item__meta">
									{{ $t('plan.minutes', {count: task.duration_min}) }}<template v-if="task.reason"> · {{ task.reason }}</template>
								</span>
							</RouterLink>
						</div>
						<p
							v-else
							class="rail-empty"
						>
							{{ $t('plan.noUnscheduled') }}
						</p>
					</section>

					<section
						v-if="plan.deferred.length || plan.errors.length"
						class="rail-block"
					>
						<template v-if="plan.deferred.length">
							<p class="rail-label">
								{{ $t('plan.deferred') }}
							</p>
							<div class="rail-list">
								<RouterLink
									v-for="task in plan.deferred"
									:key="task.task_id"
									:to="{name: 'task.detail', params: {id: task.task_id}}"
									class="rail-item"
								>
									<span class="rail-item__title">{{ task.title }}</span>
									<span class="rail-item__meta">
										{{ $t('plan.minutes', {count: task.duration_min}) }}<template v-if="task.reason"> · {{ task.reason }}</template>
									</span>
								</RouterLink>
							</div>
						</template>
						<template v-if="plan.errors.length">
							<p class="rail-label rail-label--spaced">
								{{ $t('plan.errors') }}
							</p>
							<ul class="rail-errors">
								<li
									v-for="(item, index) in plan.errors"
									:key="index"
								>
									{{ item }}
								</li>
							</ul>
						</template>
					</section>
				</aside>
			</div>
		</template>
	</div>
</template>

<script lang="ts" setup>
import {computed, onBeforeUnmount, onMounted, ref} from 'vue'
import {useI18n} from 'vue-i18n'

import {getToken} from '@/helpers/auth'

type Level = 'low' | 'medium' | 'high'

type PlanBlock = {
	task_id: number,
	title: string,
	project_id: number,
	start: string,
	end: string,
	duration_min: number,
	importance: Level,
	urgency: Level,
	source: 'planner' | 'manual',
	score: number | null,
}

type UnscheduledTask = {
	task_id: number,
	title: string,
	project_id: number,
	due_date: string | null,
	duration_min: number,
	score: number,
	reason: string,
}

type Plan = {
	date: string,
	kind: string | null,
	status: string,
	generated_at: string | null,
	window?: {
		start: string,
		end: string,
	},
	scheduled: PlanBlock[],
	unscheduled: UnscheduledTask[],
	deferred: UnscheduledTask[],
	errors: string[],
}

type LaidBlock = PlanBlock & {
	startMin: number,
	endMin: number,
	lane: number,
	lanes: number,
}

type WeekDay = {
	iso: string,
	day: string,
	weekday: string,
	isToday: boolean,
	isSelected: boolean,
	isWeekend: boolean,
}

const HOUR_HEIGHT = 64
const MIN_EVENT_HEIGHT = 32
const POLL_INTERVAL = 30_000
const AUTO_PLAN_INTERVAL = 5 * 60_000
const FALLBACK_START = 8 * 60
const FALLBACK_END = 23 * 60

const {t, locale} = useI18n({useScope: 'global'})
const plan = ref<Plan | null>(null)
const selectedDate = ref('')
const loading = ref(false)
const refreshing = ref(false)
const fetching = ref(false)
const running = ref(false)
const errorMessage = ref('')
const lastSync = ref('')
const now = ref(new Date())

let pollTimer: number | undefined
let nowTimer: number | undefined
let planTimer: number | undefined
let lastRunAt = 0

const dateLocale = computed(() => locale.value.startsWith('tr') ? 'tr-TR' : 'en-GB')
const effectiveDate = computed(() => selectedDate.value || plan.value?.date || todayIso())
const isToday = computed(() => effectiveDate.value === todayIso())

function todayIso(): string {
	return new Intl.DateTimeFormat('en-CA', {
		year: 'numeric',
		month: '2-digit',
		day: '2-digit',
		timeZone: 'Europe/Istanbul',
	}).format(new Date())
}

function parseIso(value: string): Date {
	return new Date(`${value}T12:00:00Z`)
}

function toIso(value: Date): string {
	return value.toISOString().slice(0, 10)
}

function addDays(value: Date, days: number): Date {
	const next = new Date(value)
	next.setUTCDate(next.getUTCDate() + days)
	return next
}

function weekStart(value: string): Date {
	const date = parseIso(value)
	const offset = (date.getUTCDay() + 6) % 7
	return addDays(date, -offset)
}

function formatHour(minutes: number): string {
	const hours = Math.floor(minutes / 60)
	return `${String(hours).padStart(2, '0')}:00`
}

function formatMinutes(minutes: number): string {
	const hours = Math.floor(minutes / 60)
	return `${String(hours).padStart(2, '0')}:${String(minutes % 60).padStart(2, '0')}`
}

function minutesFromIso(value: string): number {
	const match = value.match(/T(\d{2}):(\d{2})/)
	if (!match) return 0
	return Number(match[1]) * 60 + Number(match[2])
}

function minutesInIstanbul(value: Date): number {
	const parts = new Intl.DateTimeFormat('en-GB', {
		hour: '2-digit',
		minute: '2-digit',
		hourCycle: 'h23',
		timeZone: 'Europe/Istanbul',
	}).formatToParts(value)
	const hour = Number(parts.find(part => part.type === 'hour')?.value ?? 0)
	const minute = Number(parts.find(part => part.type === 'minute')?.value ?? 0)
	return hour * 60 + minute
}

function time(value: string): string {
	return new Intl.DateTimeFormat(dateLocale.value, {
		hour: '2-digit',
		minute: '2-digit',
		hourCycle: 'h23',
		timeZone: 'Europe/Istanbul',
	}).format(new Date(value))
}

function formatClock(value: Date): string {
	return new Intl.DateTimeFormat(dateLocale.value, {
		hour: '2-digit',
		minute: '2-digit',
		hourCycle: 'h23',
		timeZone: 'Europe/Istanbul',
	}).format(value)
}

const windowStart = computed(() => {
	const value = plan.value?.window?.start
	return value ? minutesFromIso(`T${value}:00`) : FALLBACK_START
})
const windowEnd = computed(() => {
	const value = plan.value?.window?.end
	return value ? minutesFromIso(`T${value}:00`) : FALLBACK_END
})
const calendarHeight = computed(() => ((windowEnd.value - windowStart.value) / 60) * HOUR_HEIGHT)
const calendarStyle = computed(() => ({
	'--hour-height': `${HOUR_HEIGHT}px`,
	height: `${calendarHeight.value}px`,
}))
const hours = computed(() => {
	const result: string[] = []
	for (let minute = windowStart.value; minute <= windowEnd.value; minute += 60) {
		result.push(formatHour(minute))
	}
	return result
})
const windowLabel = computed(() => ({
	start: formatMinutes(windowStart.value),
	end: formatMinutes(windowEnd.value),
}))
const nowMinutes = computed(() => minutesInIstanbul(now.value))
const nowTop = computed(() => {
	if (!isToday.value) return null
	if (nowMinutes.value < windowStart.value || nowMinutes.value > windowEnd.value) return null
	return ((nowMinutes.value - windowStart.value) / 60) * HOUR_HEIGHT
})

const dayParts = computed(() => {
	const date = parseIso(effectiveDate.value)
	return {
		day: new Intl.DateTimeFormat(dateLocale.value, {day: 'numeric', timeZone: 'UTC'}).format(date),
		monthYear: new Intl.DateTimeFormat(dateLocale.value, {month: 'long', year: 'numeric', timeZone: 'UTC'}).format(date),
		weekday: new Intl.DateTimeFormat(dateLocale.value, {weekday: 'long', timeZone: 'UTC'}).format(date),
	}
})

const weekDays = computed<WeekDay[]>(() => {
	const start = weekStart(effectiveDate.value)
	const today = todayIso()
	return Array.from({length: 7}, (_, index) => {
		const date = addDays(start, index)
		const iso = toIso(date)
		const dayOfWeek = date.getUTCDay()
		return {
			iso,
			day: new Intl.DateTimeFormat(dateLocale.value, {day: 'numeric', timeZone: 'UTC'}).format(date),
			weekday: new Intl.DateTimeFormat(dateLocale.value, {weekday: 'short', timeZone: 'UTC'}).format(date),
			isToday: iso === today,
			isSelected: iso === effectiveDate.value,
			isWeekend: dayOfWeek === 0 || dayOfWeek === 6,
		}
	})
})

const weekRange = computed(() => {
	const start = weekStart(effectiveDate.value)
	const end = addDays(start, 6)
	const sameMonth = start.getUTCMonth() === end.getUTCMonth()
	const startText = new Intl.DateTimeFormat(dateLocale.value, {
		day: 'numeric',
		...(sameMonth ? {} : {month: 'short'}),
		timeZone: 'UTC',
	}).format(start)
	const endText = new Intl.DateTimeFormat(dateLocale.value, {
		day: 'numeric',
		month: 'short',
		year: 'numeric',
		timeZone: 'UTC',
	}).format(end)
	return `${startText} – ${endText}`
})

const planMeta = computed(() => {
	const parts = [`${windowLabel.value.start}–${windowLabel.value.end}`]
	if (plan.value?.kind) {
		parts.push(t(`plan.kind.${plan.value.kind}`))
	}
	if (plan.value?.generated_at) {
		parts.push(t('plan.generatedAt', {time: formatClock(new Date(plan.value.generated_at))}))
	}
	return parts.join(' · ')
})

const laidOutBlocks = computed<LaidBlock[]>(() => {
	const blocks: LaidBlock[] = (plan.value?.scheduled ?? [])
		.map(block => ({
			...block,
			startMin: minutesFromIso(block.start),
			endMin: minutesFromIso(block.end),
			lane: 0,
			lanes: 1,
		}))
		.sort((left, right) => left.startMin - right.startMin || left.endMin - right.endMin)

	const result: LaidBlock[] = []
	let group: LaidBlock[] = []
	let groupEnd = -1

	const flush = (): void => {
		if (group.length === 0) return
		const laneEnds: number[] = []
		for (const block of group) {
			let lane = laneEnds.findIndex(end => end <= block.startMin)
			if (lane === -1) {
				lane = laneEnds.length
				laneEnds.push(block.endMin)
			} else {
				laneEnds[lane] = block.endMin
			}
			block.lane = lane
		}
		for (const block of group) {
			block.lanes = laneEnds.length
		}
		result.push(...group)
		group = []
		groupEnd = -1
	}

	for (const block of blocks) {
		if (group.length > 0 && block.startMin >= groupEnd) {
			flush()
		}
		group.push(block)
		groupEnd = Math.max(groupEnd, block.endMin)
	}
	flush()
	return result
})

const scheduledMinutes = computed(() => laidOutBlocks.value.reduce((total, block) => total + block.duration_min, 0))
const freeMinutes = computed(() => Math.max(0, windowEnd.value - windowStart.value - scheduledMinutes.value))
const nextBlock = computed<PlanBlock | null>(() => {
	const blocks = [...(plan.value?.scheduled ?? [])].sort((left, right) => minutesFromIso(left.start) - minutesFromIso(right.start))
	if (isToday.value) {
		return blocks.find(block => minutesFromIso(block.end) > nowMinutes.value) ?? null
	}
	return blocks[0] ?? null
})

function eventStyle(block: LaidBlock): Record<string, string> {
	const top = ((block.startMin - windowStart.value) / 60) * HOUR_HEIGHT
	const height = Math.max(((block.endMin - block.startMin) / 60) * HOUR_HEIGHT, MIN_EVENT_HEIGHT)
	const left = block.lanes > 1 ? `calc(${(block.lane / block.lanes) * 100}% + 2px)` : '2px'
	const width = block.lanes > 1 ? `calc(${100 / block.lanes}% - 4px)` : 'calc(100% - 4px)'
	return {
		top: `${top}px`,
		height: `${height}px`,
		left,
		width,
	}
}

function eventHeight(block: LaidBlock): number {
	return Math.max(((block.endMin - block.startMin) / 60) * HOUR_HEIGHT, MIN_EVENT_HEIGHT)
}

function isPast(block: PlanBlock): boolean {
	return new Date(block.end) < now.value
}

function isCurrent(block: PlanBlock): boolean {
	return new Date(block.start) <= now.value && now.value < new Date(block.end)
}

function eventClasses(block: LaidBlock): Record<string, boolean> {
	return {
		[`is-${block.importance}`]: true,
		'is-manual': block.source === 'manual',
		'is-planned': block.source !== 'manual',
		'is-short': eventHeight(block) < 52,
		'is-past': isPast(block),
		'is-now': isCurrent(block),
	}
}

function eventAria(block: PlanBlock): string {
	const source = block.source === 'manual' ? t('plan.manual') : t('plan.planned')
	return `${t('plan.minutes', {count: block.duration_min})} · ${block.title} · ${time(block.start)}–${time(block.end)} · ${source}`
}

function selectDate(value: string): void {
	selectedDate.value = value
	void fetchPlan(value)
}

function shiftDate(days: number): void {
	selectDate(toIso(addDays(parseIso(effectiveDate.value), days)))
}

function goToday(): void {
	selectDate(todayIso())
}

function onDateChange(): void {
	if (selectedDate.value) {
		void fetchPlan(selectedDate.value)
	}
}

async function fetchPlan(date?: string, options: {autoRun?: boolean} = {}): Promise<void> {
	const token = getToken()
	if (!token) {
		errorMessage.value = t('plan.sessionMissing')
		return
	}
	if (fetching.value) return

	fetching.value = true
	refreshing.value = Boolean(plan.value)
	if (!plan.value) {
		loading.value = true
	}
	errorMessage.value = ''
	try {
		const query = date ? `?date=${encodeURIComponent(date)}` : ''
		const response = await fetch(`/capture/api/plan${query}`, {
			headers: {Authorization: `Bearer ${token}`},
		})
		const contentType = response.headers.get('content-type') ?? ''
		const body = contentType.includes('application/json')
			? await response.json().catch(() => null)
			: null
		if (!response.ok || !body || !Array.isArray(body.scheduled)) {
			throw new Error(body?.error?.message ?? t('plan.loadFailed'))
		}
		plan.value = body as Plan
		selectedDate.value = plan.value.date
		lastSync.value = formatClock(new Date())
		if (options.autoRun !== false && plan.value.status === 'empty') {
			void runPlan(plan.value.date)
		}
	} catch (error) {
		errorMessage.value = error instanceof Error ? error.message : t('plan.loadFailed')
	} finally {
		loading.value = false
		refreshing.value = false
		fetching.value = false
	}
}

async function runPlan(date?: string): Promise<void> {
	const token = getToken()
	if (!token) {
		errorMessage.value = t('plan.sessionMissing')
		return
	}
	if (running.value) return

	running.value = true
	errorMessage.value = ''
	try {
		const response = await fetch('/capture/api/plan/run', {
			method: 'POST',
			headers: {
				Authorization: `Bearer ${token}`,
				'Content-Type': 'application/json',
			},
			body: JSON.stringify({date: date || selectedDate.value || undefined}),
		})
		const contentType = response.headers.get('content-type') ?? ''
		const body = contentType.includes('application/json')
			? await response.json().catch(() => null)
			: null
		if (!response.ok || !body || !Array.isArray(body.scheduled)) {
			throw new Error(body?.error?.message ?? t('plan.runFailed'))
		}
		plan.value = body as Plan
		selectedDate.value = plan.value.date
		lastSync.value = formatClock(new Date())
		lastRunAt = Date.now()
	} catch (error) {
		errorMessage.value = error instanceof Error ? error.message : t('plan.runFailed')
	} finally {
		running.value = false
	}
}

function updateNow(): void {
	now.value = new Date()
}

function onVisibilityChange(): void {
	if (document.hidden) return
	void fetchPlan(selectedDate.value || undefined)
	if (Date.now() - lastRunAt > AUTO_PLAN_INTERVAL) {
		void runPlan(selectedDate.value || undefined)
	}
}

function onFocus(): void {
	void fetchPlan(selectedDate.value || undefined)
}

onMounted(() => {
	updateNow()
	void fetchPlan()
	pollTimer = window.setInterval(() => {
		if (!document.hidden) {
			void fetchPlan(selectedDate.value || undefined)
		}
	}, POLL_INTERVAL)
	nowTimer = window.setInterval(updateNow, POLL_INTERVAL)
	planTimer = window.setInterval(() => {
		if (!document.hidden) {
			void runPlan(selectedDate.value || undefined)
		}
	}, AUTO_PLAN_INTERVAL)
	document.addEventListener('visibilitychange', onVisibilityChange)
	window.addEventListener('focus', onFocus)
})

onBeforeUnmount(() => {
	if (pollTimer) window.clearInterval(pollTimer)
	if (nowTimer) window.clearInterval(nowTimer)
	if (planTimer) window.clearInterval(planTimer)
	document.removeEventListener('visibilitychange', onVisibilityChange)
	window.removeEventListener('focus', onFocus)
})
</script>

<style lang="scss" scoped>
$ink: var(--text);
$muted: var(--text-muted);
$rule: var(--grey-200);
$rule-strong: var(--grey-300);
$paper: var(--white);
$accent: var(--primary);
$now: var(--danger);

.visually-hidden {
	position: absolute;
	inline-size: 1px;
	block-size: 1px;
	padding: 0;
	margin: -1px;
	overflow: hidden;
	clip: rect(0, 0, 0, 0);
	white-space: nowrap;
	border: 0;
}

.plan-view {
	max-inline-size: 78rem;
	margin-inline: auto;
	padding-block-end: 3rem;
}

.plan-head {
	display: flex;
	justify-content: space-between;
	align-items: flex-start;
	gap: 1.5rem 2rem;
	flex-wrap: wrap;
	padding-block-end: 1.25rem;
	border-block-end: 1px solid $rule;

	&__intro {
		min-inline-size: 0;
	}

	&__eyebrow {
		margin: 0 0 .5rem;
		font-size: .72rem;
		font-weight: 700;
		letter-spacing: .16em;
		text-transform: uppercase;
		color: $muted;
	}

	&__date {
		display: flex;
		align-items: baseline;
		gap: .75rem;
		margin: 0;
		font-family: Quicksand, 'Open Sans', sans-serif;
		font-weight: 600;
		line-height: 1;
		color: $ink;
	}

	&__day-number {
		font-size: clamp(3rem, 7vw, 4.75rem);
		font-variant-numeric: tabular-nums;
		letter-spacing: -.04em;
	}

	&__date-text {
		display: flex;
		flex-direction: column;
		gap: .3rem;
	}

	&__month {
		font-size: clamp(1.1rem, 2.4vw, 1.5rem);
		letter-spacing: -.01em;
	}

	&__weekday {
		font-family: $vikunja-font;
		font-size: .8rem;
		font-weight: 400;
		letter-spacing: .12em;
		text-transform: uppercase;
		color: $muted;
	}

	&__window {
		margin: .85rem 0 0;
		font-size: .86rem;
		color: $muted;
		font-variant-numeric: tabular-nums;
	}

	&__side {
		display: flex;
		flex-direction: column;
		align-items: flex-end;
		gap: .9rem;
	}

	&__live {
		display: inline-flex;
		align-items: center;
		gap: .45rem;
		min-block-size: 1.6rem;
		padding: .28rem .7rem;
		border: 1px solid $rule;
		border-radius: 999px;
		background: $paper;
		font-size: .72rem;
		font-weight: 600;
		letter-spacing: .08em;
		text-transform: uppercase;
		color: $muted;
		white-space: nowrap;

		&.is-busy {
			color: $accent;
			border-color: $accent;
		}
	}

	&__live-dot {
		inline-size: .45rem;
		block-size: .45rem;
		border-radius: 50%;
		background: var(--success);
		box-shadow: 0 0 0 3px hsl(var(--success-h), var(--success-s), var(--success-l), .16);
		animation: live-pulse 2.4s ease-in-out infinite;
	}

	&__live.is-busy &__live-dot {
		background: $accent;
		animation-duration: 1s;
	}

	&__live-time {
		font-weight: 400;
		letter-spacing: .02em;
		text-transform: none;
		color: $muted;
		padding-inline-start: .35rem;
		border-inline-start: 1px solid $rule;
	}

	&__controls {
		display: flex;
		align-items: center;
		gap: .35rem;
		flex-wrap: wrap;
		justify-content: flex-end;
	}
}

.icon-button,
.today-button,
.date-picker {
	border: 1px solid $rule;
	background: $paper;
	color: $ink;
	font-family: inherit;
	cursor: pointer;
	transition: border-color .15s ease, background-color .15s ease, color .15s ease;

	&:hover {
		border-color: $rule-strong;
	}

	&:focus-visible {
		outline: 2px solid $accent;
		outline-offset: 2px;
	}
}

.icon-button {
	display: inline-flex;
	align-items: center;
	justify-content: center;
	inline-size: 2.25rem;
	block-size: 2.25rem;
	padding: 0;
	border-radius: 999px;

	svg {
		inline-size: 1.1rem;
		block-size: 1.1rem;
		fill: none;
		stroke: currentcolor;
		stroke-width: 1.8;
		stroke-linecap: round;
		stroke-linejoin: round;
	}
}

.today-button {
	min-block-size: 2.25rem;
	padding-inline: .95rem;
	border-radius: 999px;
	font-size: .8rem;
	font-weight: 600;

	&:disabled {
		opacity: .45;
		cursor: default;
	}
}

.date-picker {
	position: relative;
	display: inline-flex;
	align-items: center;
	justify-content: center;
	inline-size: 2.25rem;
	block-size: 2.25rem;
	border-radius: 999px;
	overflow: hidden;

	svg {
		inline-size: 1.05rem;
		block-size: 1.05rem;
		fill: none;
		stroke: currentcolor;
		stroke-width: 1.7;
		stroke-linecap: round;
		stroke-linejoin: round;
		pointer-events: none;
	}

	input {
		position: absolute;
		inset: 0;
		inline-size: 100%;
		block-size: 100%;
		opacity: 0;
		cursor: pointer;
	}
}

.week-strip {
	display: flex;
	align-items: flex-end;
	justify-content: space-between;
	gap: 1.5rem;
	flex-wrap: wrap;
	padding-block: 1rem 1.4rem;

	&__range {
		margin: 0;
		font-size: .78rem;
		font-weight: 600;
		letter-spacing: .1em;
		text-transform: uppercase;
		color: $muted;
		white-space: nowrap;
	}

	&__days {
		display: grid;
		grid-template-columns: repeat(7, minmax(2.5rem, 1fr));
		gap: .25rem;
		inline-size: min(100%, 34rem);
	}

	&__day {
		position: relative;
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: .2rem;
		min-block-size: 3.5rem;
		padding: .45rem .25rem .5rem;
		border: 1px solid transparent;
		border-radius: .6rem;
		background: transparent;
		color: $ink;
		font-family: inherit;
		cursor: pointer;
		transition: background-color .15s ease, border-color .15s ease, color .15s ease;

		&:hover {
			background: var(--grey-100);
		}

		&:focus-visible {
			outline: 2px solid $accent;
			outline-offset: 2px;
		}

		&.is-weekend {
			color: $muted;
		}

		&.is-selected {
			border-color: $ink;
			background: $paper;

			.week-strip__number {
				font-weight: 700;
			}
		}

		&.is-today:not(.is-selected) {
			background: var(--grey-100);
		}
	}

	&__weekday {
		font-size: .62rem;
		font-weight: 700;
		letter-spacing: .1em;
		text-transform: uppercase;
		color: $muted;
	}

	&__number {
		font-size: 1.05rem;
		font-variant-numeric: tabular-nums;
		line-height: 1.1;
	}

	&__today {
		inline-size: .3rem;
		block-size: .3rem;
		border-radius: 50%;
		background: $accent;
	}
}

.plan-alert {
	margin-block: .5rem 1rem;
	padding: .7rem .9rem;
	border-inline-start: 3px solid $now;
	background: var(--danger-light);
	color: $ink;
	font-size: .86rem;
	border-radius: 0 $radius $radius 0;
}

.plan-loading {
	display: flex;
	align-items: center;
	gap: .6rem;
	padding-block: 3rem;
	color: $muted;

	&__spinner {
		inline-size: 1rem;
		block-size: 1rem;
		border: 2px solid $rule;
		border-block-start-color: $accent;
		border-radius: 50%;
		animation: spin .8s linear infinite;
	}
}

.plan-grid {
	display: grid;
	grid-template-columns: minmax(0, 1fr) 19rem;
	gap: 1.5rem;
	align-items: start;
}

.plan-calendar {
	min-inline-size: 0;
	padding: 1rem 1rem .75rem;
	border: 1px solid $rule;
	border-radius: .9rem;
	background: $paper;
	box-shadow: var(--shadow-sm);
}

.calendar {
	display: grid;
	grid-template-columns: 3.75rem minmax(0, 1fr);

	&__gutter {
		position: relative;
		display: flex;
		flex-direction: column;
		padding-inline-end: .5rem;
	}

	&__hour {
		display: block;
		block-size: var(--hour-height);
		font-size: .7rem;
		font-weight: 600;
		font-variant-numeric: tabular-nums;
		line-height: 1;
		color: $muted;
		transform: translateY(-.35em);
	}

	&__canvas {
		position: relative;
		border-inline-start: 1px solid $rule;
	}

	&__grid {
		position: absolute;
		inset: 0;
		background-image:
			repeating-linear-gradient(to bottom, $rule 0 1px, transparent 1px var(--hour-height)),
			repeating-linear-gradient(to bottom, transparent 0 calc(var(--hour-height) / 2), var(--grey-100) calc(var(--hour-height) / 2) calc(var(--hour-height) / 2 + 1px), transparent calc(var(--hour-height) / 2 + 1px) var(--hour-height));
		pointer-events: none;
	}

	&__now {
		position: absolute;
		inset-inline: -.35rem 0;
		z-index: 4;
		display: flex;
		align-items: center;
		gap: .4rem;
		pointer-events: none;
		transform: translateY(-50%);

		&::after {
			content: '';
			flex: 1;
			block-size: 2px;
			background: $now;
		}
	}

	&__now-dot {
		inline-size: .55rem;
		block-size: .55rem;
		border-radius: 50%;
		background: $now;
		box-shadow: 0 0 0 3px hsl(var(--danger-h), var(--danger-s), var(--danger-l), .2);
	}

	&__now-label {
		font-size: .64rem;
		font-weight: 700;
		letter-spacing: .08em;
		text-transform: uppercase;
		color: $now;
		white-space: nowrap;
	}

	&__empty {
		position: absolute;
		inset-inline: 0;
		inset-block-start: 2rem;
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: .35rem;
		padding-inline: 1.5rem;
		text-align: center;
		pointer-events: none;
	}

	&__empty-title {
		margin: 0;
		font-family: Quicksand, 'Open Sans', sans-serif;
		font-size: 1.05rem;
		font-weight: 600;
		color: $ink;
	}

	&__empty-hint {
		margin: 0;
		max-inline-size: 24rem;
		font-size: .82rem;
		line-height: 1.5;
		color: $muted;
	}

	&__legend {
		display: flex;
		align-items: center;
		gap: 1rem;
		flex-wrap: wrap;
		margin-block-start: .85rem;
		padding-block-start: .7rem;
		border-block-start: 1px solid $rule;
		font-size: .7rem;
		color: $muted;

		span {
			display: inline-flex;
			align-items: center;
			gap: .35rem;
		}
	}
}

.legend-swatch {
	inline-size: .55rem;
	block-size: .55rem;
	border-radius: 2px;

	&.is-manual {
		border: 1.5px dashed var(--warning);
		background: transparent;
	}

	&.is-planned {
		background: $accent;
	}

	&.is-high {
		background: var(--danger);
	}
}

.event {
	position: absolute;
	z-index: 2;
	display: flex;
	flex-direction: column;
	gap: .1rem;
	overflow: hidden;
	padding: .35rem .55rem .4rem .7rem;
	border: 1px solid $rule;
	border-inline-start: 3px solid $accent;
	border-radius: .4rem;
	background: $paper;
	color: $ink;
	text-decoration: none;
	box-shadow: 0 1px 2px hsl(var(--grey-500-hsl), .08);
	transition: box-shadow .15s ease, transform .15s ease, border-color .15s ease;

	&:hover {
		box-shadow: var(--shadow-md);
		transform: translateY(-1px);
	}

	&:focus-visible {
		outline: 2px solid $accent;
		outline-offset: 2px;
	}

	&.is-high {
		border-inline-start-color: var(--danger);
	}

	&.is-medium {
		border-inline-start-color: var(--warning);
	}

	&.is-low {
		border-inline-start-color: var(--success);
	}

	&.is-manual {
		border-style: dashed;
		border-inline-start-style: solid;
		background: var(--grey-100);
	}

	&.is-now {
		border-color: $accent;
		box-shadow: 0 0 0 2px hsl(var(--primary-h), var(--primary-s), var(--primary-l), .18);
	}

	&.is-past {
		opacity: .55;
	}

	&.is-short {
		flex-direction: row;
		align-items: baseline;
		gap: .45rem;
		padding-block: .3rem;

		.event__meta {
			display: none;
		}

		.event__title {
			overflow: hidden;
			text-overflow: ellipsis;
			white-space: nowrap;
		}
	}

	&__time {
		font-size: .66rem;
		font-weight: 700;
		font-variant-numeric: tabular-nums;
		letter-spacing: .02em;
		color: $muted;
		white-space: nowrap;
	}

	&__title {
		display: -webkit-box;
		overflow: hidden;
		font-family: Quicksand, 'Open Sans', sans-serif;
		font-size: .86rem;
		font-weight: 600;
		line-height: 1.25;
		color: $ink;
		-webkit-box-orient: vertical;
		-webkit-line-clamp: 2;
	}

	&__meta {
		display: flex;
		align-items: center;
		gap: .45rem;
		flex-wrap: wrap;
		font-size: .64rem;
		line-height: 1.2;
		color: $muted;
	}

	&__source {
		padding-inline-start: .45rem;
		border-inline-start: 1px solid $rule;
	}

	&__importance {
		display: inline-flex;
		align-items: center;
		gap: .25rem;

		&.is-high .event__dot {
			background: var(--danger);
		}

		&.is-medium .event__dot {
			background: var(--warning);
		}

		&.is-low .event__dot {
			background: var(--success);
		}
	}

	&__dot {
		inline-size: .4rem;
		block-size: .4rem;
		border-radius: 50%;
	}
}

.plan-rail {
	display: flex;
	flex-direction: column;
	gap: .9rem;
	min-inline-size: 0;
}

.rail-block {
	padding: .9rem 1rem;
	border: 1px solid $rule;
	border-radius: .75rem;
	background: $paper;

	&--stats {
		background: var(--grey-100);
		border-color: transparent;
	}
}

.rail-label {
	margin: 0 0 .6rem;
	font-size: .68rem;
	font-weight: 700;
	letter-spacing: .12em;
	text-transform: uppercase;
	color: $muted;

	&--spaced {
		margin-block-start: 1rem;
	}
}

.rail-stats {
	display: grid;
	grid-template-columns: 1fr 1fr;
	gap: .75rem;

	div {
		display: flex;
		flex-direction: column;
		gap: .15rem;
		min-inline-size: 0;
	}

	strong {
		font-family: Quicksand, 'Open Sans', sans-serif;
		font-size: 1.05rem;
		font-weight: 600;
		color: $ink;
	}

	span {
		font-size: .72rem;
		color: $muted;
	}
}

.next-card {
	display: flex;
	flex-direction: column;
	gap: .2rem;
	padding: .6rem .7rem;
	border-inline-start: 3px solid $accent;
	border-radius: .35rem;
	background: var(--grey-100);
	color: inherit;
	text-decoration: none;

	&:hover .next-card__title {
		text-decoration: underline;
	}

	&:focus-visible {
		outline: 2px solid $accent;
		outline-offset: 2px;
	}

	&__time {
		font-size: .68rem;
		font-weight: 700;
		font-variant-numeric: tabular-nums;
		color: $accent;
	}

	&__title {
		font-family: Quicksand, 'Open Sans', sans-serif;
		font-size: .9rem;
		font-weight: 600;
		color: $ink;
	}

	&__meta {
		font-size: .68rem;
		color: $muted;
	}
}

.rail-list {
	display: flex;
	flex-direction: column;
	gap: .15rem;
}

.rail-item {
	display: flex;
	flex-direction: column;
	gap: .1rem;
	padding: .45rem .55rem;
	border-radius: .35rem;
	color: inherit;
	text-decoration: none;

	&:hover {
		background: var(--grey-100);
	}

	&:focus-visible {
		outline: 2px solid $accent;
		outline-offset: 2px;
	}

	&__title {
		font-size: .82rem;
		font-weight: 500;
		line-height: 1.35;
		color: $ink;
	}

	&__meta {
		font-size: .68rem;
		line-height: 1.35;
		color: $muted;
	}
}

.rail-empty {
	margin: 0;
	font-size: .78rem;
	line-height: 1.45;
	color: $muted;
}

.rail-errors {
	margin: 0;
	padding-inline-start: 1rem;
	font-size: .75rem;
	line-height: 1.45;
	color: var(--danger-dark);
}

@keyframes live-pulse {
	0%, 100% {
		opacity: 1;
	}

	50% {
		opacity: .35;
	}
}

@keyframes spin {
	to {
		transform: rotate(360deg);
	}
}

@media (width <= 960px) {
	.plan-grid {
		grid-template-columns: minmax(0, 1fr);
	}

	.plan-rail {
		display: grid;
		grid-template-columns: repeat(2, minmax(0, 1fr));
		align-items: start;
	}
}

@media (width <= 700px) {
	.plan-head {
		&__side {
			align-items: flex-start;
			inline-size: 100%;
		}

		&__controls {
			justify-content: flex-start;
		}
	}

	.week-strip {
		align-items: flex-start;
		flex-direction: column;
		gap: .65rem;

		&__days {
			inline-size: 100%;
		}
	}

	.plan-calendar {
		padding: .75rem .65rem .65rem;
		border-radius: .65rem;
	}

	.calendar {
		grid-template-columns: 2.9rem minmax(0, 1fr);

		&__gutter {
			padding-inline-end: .4rem;
		}
	}

	.event {
		padding-inline: .45rem .5rem;

		&__meta {
			gap: .35rem;
		}
	}

	.plan-rail {
		grid-template-columns: minmax(0, 1fr);
	}
}

@media (width <= 460px) {
	.plan-head__date {
		gap: .55rem;
	}

	.plan-head__day-number {
		font-size: 2.8rem;
	}

	.week-strip__day {
		min-block-size: 3.2rem;
		padding-inline: .1rem;
	}

	.week-strip__weekday {
		font-size: .56rem;
	}

	.calendar {
		grid-template-columns: 2.55rem minmax(0, 1fr);
	}

	.event {
		&__title {
			font-size: .8rem;
		}

		&__meta {
			font-size: .6rem;
		}
	}
}

@media (prefers-reduced-motion: reduce) {
	.plan-head__live-dot,
	.plan-loading__spinner {
		animation: none;
	}

	.event,
	.week-strip__day,
	.icon-button,
	.today-button,
	.date-picker {
		transition: none;
	}
}
</style>
