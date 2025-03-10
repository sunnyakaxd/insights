<script setup lang="ts">
import { watchDebounced } from '@vueuse/core'
import { AlertTriangle } from 'lucide-vue-next'
import { computed, inject, ref } from 'vue'
import { Chart, getCachedChart } from '../charts/chart'
import ChartRenderer from '../charts/components/ChartRenderer.vue'
import { WorkbookDashboardChart, WorkbookDashboardItem } from '../types/workbook.types'
import { Dashboard } from './dashboard'
import DashboardItemActions from './DashboardItemActions.vue'

const props = defineProps<{
	index: number
	item: WorkbookDashboardItem
}>()

const dashboard = inject('dashboard') as Dashboard

const chart = computed(() => {
	if (props.item.type != 'chart') return null
	const item = props.item as WorkbookDashboardChart
	return getCachedChart(item.chart) as Chart
})

if (!chart.value?.dataQuery.result.executedSQL) {
	dashboard.refreshChart(props.item.chart)
}

watchDebounced(
	() => chart.value?.doc.config.order_by,
	() => dashboard.refreshChart(props.item.chart),
	{
		deep: true,
		debounce: 500,
	}
)

let timer: any
const wasDragging = ref(false)
const showPopover = ref(false)
const popoverDelay = 300
document.addEventListener('mousemove', (event) => {
	// if mouse moves while the button is pressed, it's dragging
	// once the button is released, it's not dragging
	// if not dragging then show popover after delay
	if (wasDragging.value && event.buttons == 0) {
		clearTimeout(timer)
		timer = setTimeout(() => (showPopover.value = true), popoverDelay)
		wasDragging.value = false
	}
	if (event.buttons == 1) {
		wasDragging.value = true
		showPopover.value = false
		clearTimeout(timer)
	}
})
</script>

<template>
	<div class="relative h-full w-full p-2 [&>div:first-child]:h-full">
		<Popover
			class="h-full"
			:show="dashboard.editing && dashboard.isActiveItem(index) && showPopover"
			placement="top-start"
		>
			<template #target>
				<div
					class="flex h-full w-full items-center rounded"
					:class="[
						dashboard.editing && dashboard.isActiveItem(index)
							? 'outline outline-gray-700'
							: '',
					]"
					@click="dashboard.setActiveItem(index)"
				>
					<div
						class="h-full w-full"
						:class="dashboard.editing ? 'pointer-events-none' : ''"
					>
						<ChartRenderer
							v-if="chart"
							:title="chart.doc.title"
							:chart_type="chart.doc.chart_type"
							:config="chart.doc.config"
							:operations="chart.doc.operations"
							:use_live_connection="chart.doc.use_live_connection"
							:result="chart.dataQuery.result"
							:loading="chart.dataQuery.executing"
						/>

						<div
							v-else
							class="flex h-full flex-1 flex-col items-center justify-center rounded border"
						>
							<AlertTriangle class="h-8 w-8 text-gray-500" stroke-width="1" />
							<p class="text-p-base text-gray-500">Chart not found</p>
						</div>
					</div>
				</div>
			</template>
			<template #body>
				<DashboardItemActions
					:dashboard="dashboard"
					:item-index="index"
					:item="dashboard.doc.items[index]"
				/>
			</template>
		</Popover>
	</div>
</template>
