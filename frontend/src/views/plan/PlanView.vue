<template>
	<div class="content plan-view">
		<header class="plan-view__header">
			<div>
				<h1>{{ $t('plan.title') }}</h1>
				<p class="has-text-grey">
					{{ $t('plan.subtitle') }}
				</p>
			</div>
			<div class="plan-view__controls">
				<input
					v-model="selectedDate"
					class="input"
					type="date"
					:aria-label="$t('plan.date')"
					@change="fetchPlan(selectedDate)"
				>
				<XButton
					variant="primary"
					:shadow="false"
					:loading="running"
					:disabled="running"
					@click="runPlan"
				>
					{{ $t('plan.refresh') }}
				</XButton>
			</div>
		</header>

		<p
			v-if="errorMessage"
			class="has-text-danger"
			role="alert"
		>
			{{ errorMessage }}
		</p>

		<div
			v-if="loading"
			class="plan-view__loading"
		>
			{{ $t('plan.loading') }}
		</div>

		<template v-else-if="plan">
			<div class="plan-view__meta">
				<strong>{{ formatPlanDate(plan.date) }}</strong>
				<span v-if="plan.kind">{{ $t(`plan.kind.${plan.kind}`) }}</span>
				<span v-if="plan.generated_at">{{ time(plan.generated_at) }}</span>
			</div>

			<section class="plan-view__section">
				<h2>{{ $t('plan.scheduled') }}</h2>
				<p
					v-if="plan.scheduled.length === 0"
					class="has-text-grey"
				>
					{{ $t('plan.empty') }}
				</p>
				<div class="plan-view__timeline">
					<RouterLink
						v-for="block in plan.scheduled"
						:key="`${block.task_id}-${block.start}`"
						:to="{name: 'task.detail', params: {id: block.task_id}}"
						class="plan-view__block"
						:class="`is-${block.importance}`"
					>
						<div class="plan-view__time">
							{{ time(block.start) }}–{{ time(block.end) }}
						</div>
						<div class="plan-view__body">
							<div class="plan-view__title">
								{{ block.title }}
							</div>
							<div class="plan-view__meta-line">
								<span>{{ $t('plan.minutes', {count: block.duration_min}) }}</span>
								<span v-if="block.source === 'manual'">{{ $t('plan.manual') }}</span>
								<span v-else>{{ $t('plan.planned') }}</span>
								<span>{{ $t(`plan.importance.${block.importance}`) }}</span>
								<span>{{ $t(`plan.urgency.${block.urgency}`) }}</span>
							</div>
						</div>
					</RouterLink>
				</div>
			</section>

			<section
				v-if="plan.unscheduled.length > 0"
				class="plan-view__section"
			>
				<h2>{{ $t('plan.unscheduled') }}</h2>
				<div class="plan-view__unscheduled">
					<RouterLink
						v-for="task in plan.unscheduled"
						:key="task.task_id"
						:to="{name: 'task.detail', params: {id: task.task_id}}"
						class="plan-view__unscheduled-item"
					>
						<span>{{ task.title }}</span>
						<span class="has-text-grey">
							{{ $t('plan.minutes', {count: task.duration_min}) }}
						</span>
					</RouterLink>
				</div>
			</section>

			<section
				v-if="plan.deferred.length > 0"
				class="plan-view__section"
			>
				<h2>{{ $t('plan.deferred') }}</h2>
				<div class="plan-view__unscheduled">
					<RouterLink
						v-for="task in plan.deferred"
						:key="task.task_id"
						:to="{name: 'task.detail', params: {id: task.task_id}}"
						class="plan-view__unscheduled-item"
					>
						<span>{{ task.title }}</span>
						<span class="has-text-grey">
							{{ $t('plan.minutes', {count: task.duration_min}) }}
						</span>
					</RouterLink>
				</div>
			</section>

			<section
				v-if="plan.errors.length > 0"
				class="plan-view__section"
			>
				<h2>{{ $t('plan.errors') }}</h2>
				<ul>
					<li
						v-for="(item, index) in plan.errors"
						:key="index"
					>
						{{ item }}
					</li>
				</ul>
			</section>
		</template>
	</div>
</template>

<script lang="ts" setup>
import {onMounted, ref} from 'vue'
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

type DeferredTask = UnscheduledTask

type Plan = {
	date: string,
	kind: string | null,
	status: string,
	generated_at: string | null,
	scheduled: PlanBlock[],
	unscheduled: UnscheduledTask[],
	deferred: DeferredTask[],
	errors: string[],
}

const {t} = useI18n({useScope: 'global'})
const plan = ref<Plan | null>(null)
const selectedDate = ref('')
const loading = ref(false)
const running = ref(false)
const errorMessage = ref('')


function time(value: string): string {
	return new Intl.DateTimeFormat('tr-TR', {
		hour: '2-digit',
		minute: '2-digit',
		timeZone: 'Europe/Istanbul',
	}).format(new Date(value))
}

function formatPlanDate(value: string): string {
	return new Intl.DateTimeFormat('tr-TR', {
		dateStyle: 'full',
		timeZone: 'Europe/Istanbul',
	}).format(new Date(`${value}T12:00:00+03:00`))
}

async function fetchPlan(date?: string): Promise<void> {
	const token = getToken()
	if (!token) {
		errorMessage.value = t('plan.sessionMissing')
		return
	}

	loading.value = true
	errorMessage.value = ''
	try {
		const query = date ? `?date=${encodeURIComponent(date)}` : ''
		const response = await fetch(`/capture/api/plan${query}`, {
			headers: {Authorization: `Bearer ${token}`},
		})
		const body = await response.json().catch(() => ({}))
		if (!response.ok) {
			throw new Error(body?.error?.message ?? `HTTP ${response.status}`)
		}
		plan.value = body as Plan
		selectedDate.value = plan.value.date
	} catch (error) {
		errorMessage.value = error instanceof Error ? error.message : t('plan.loadFailed')
	} finally {
		loading.value = false
	}
}

async function runPlan(): Promise<void> {
	const token = getToken()
	if (!token) {
		errorMessage.value = t('plan.sessionMissing')
		return
	}

	running.value = true
	errorMessage.value = ''
	try {
		const response = await fetch('/capture/api/plan/run', {
			method: 'POST',
			headers: {
				Authorization: `Bearer ${token}`,
				'Content-Type': 'application/json',
			},
			body: JSON.stringify({date: selectedDate.value || undefined}),
		})
		const body = await response.json().catch(() => ({}))
		if (!response.ok) {
			throw new Error(body?.error?.message ?? `HTTP ${response.status}`)
		}
		plan.value = body as Plan
		selectedDate.value = plan.value.date
	} catch (error) {
		errorMessage.value = error instanceof Error ? error.message : t('plan.runFailed')
	} finally {
		running.value = false
	}
}

onMounted(() => {
	void fetchPlan()
})
</script>

<style lang="scss" scoped>
.plan-view {
	max-inline-size: 56rem;
	margin-inline: auto;

	&__header {
		display: flex;
		justify-content: space-between;
		align-items: flex-start;
		gap: 1rem;
		flex-wrap: wrap;

		h1 {
			margin-block-end: 0.25rem;
		}
	}

	&__controls {
		display: flex;
		gap: 0.5rem;
		align-items: center;
		flex-wrap: wrap;
	}

	&__meta {
		display: flex;
		gap: 0.75rem;
		align-items: center;
		flex-wrap: wrap;
		color: var(--grey-500);
		margin-block: 0.5rem 1.5rem;
	}

	&__section {
		margin-block-end: 1.75rem;

		h2 {
			font-size: 1.1rem;
			margin-block-end: 0.75rem;
		}
	}

	&__timeline {
		display: flex;
		flex-direction: column;
		gap: 0.45rem;
	}

	&__block {
		display: grid;
		grid-template-columns: 7.5rem minmax(0, 1fr);
		gap: 0.75rem;
		padding: 0.65rem 0.85rem;
		background: var(--white);
		border-inline-start: 4px solid var(--grey-300);
		border-radius: $radius;
		box-shadow: var(--shadow-sm);
		color: inherit;

		&:hover {
			box-shadow: var(--shadow-md);
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
	}

	&__time {
		font-variant-numeric: tabular-nums;
		font-weight: 600;
	}

	&__title {
		font-weight: 600;
	}

	&__meta-line {
		display: flex;
		gap: 0.75rem;
		flex-wrap: wrap;
		font-size: 0.85rem;
		color: var(--grey-500);
	}

	&__unscheduled {
		display: flex;
		flex-direction: column;
		gap: 0.35rem;
	}

	&__unscheduled-item {
		display: flex;
		justify-content: space-between;
		gap: 1rem;
		padding: 0.5rem 0.75rem;
		background: var(--grey-100);
		border-radius: $radius;
		color: inherit;
	}

	&__loading {
		color: var(--grey-500);
		padding-block: 2rem;
	}
}
</style>
