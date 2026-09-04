<script setup lang="ts">
import { watchDebounced } from '@vueuse/core'
import Fuse from 'fuse.js'
import { computed, ref } from 'vue'
import OptionSelectGroup from '~/components/OptionSelectGroup.vue'
import RuleLevelIcon from '~/components/RuleLevelIcon.vue'
import RuleList from '~/components/RuleList.vue'
import { getPluginColor } from '~/composables/color'
import { payload } from '~/composables/payload'
import { bpSm, filtersRules as filters, stateStorage } from '~/composables/state'

const rules = computed(() => Object.values(payload.value.rules))
const pluginNames = computed(() => Array.from(new Set(rules.value.map(i => i.plugin))).filter(Boolean))

const conditionalFiltered = computed(() => {
  let conditional = rules.value

  if (filters.plugin) {
    conditional = conditional
      .filter(rule => rule.plugin === filters.plugin)
  }

  if (filters.fixable != null) {
    conditional = conditional
      .filter(rule => !!rule.fixable === filters.fixable)
  }

  switch (filters.state) {
    case 'using':
      conditional = conditional.filter(rule => payload.value.ruleToState.get(rule.name))
      break
    case 'unused':
      conditional = conditional.filter(rule => !payload.value.ruleToState.get(rule.name))
      break
    case 'overloads':
      conditional = conditional.filter(rule => (payload.value.ruleToState.get(rule.name)?.length || 0) > 1)
      break
    case 'error':
      conditional = conditional.filter(rule => payload.value.ruleToState.get(rule.name)?.some(i => i.level === 'error'))
      break
    case 'warn':
      conditional = conditional.filter(rule => payload.value.ruleToState.get(rule.name)?.some(i => i.level === 'warn'))
      break
    case 'off':
      conditional = conditional.filter(rule => payload.value.ruleToState.get(rule.name)?.some(i => i.level === 'off'))
      break
    case 'off-only':
      conditional = conditional.filter((rule) => {
        const states = payload.value.ruleToState.get(rule.name)
        if (!states?.length)
          return false
        return states.every(i => i.level === 'off')
      })
      break
  }

  switch (filters.status) {
    case 'active':
      conditional = conditional.filter(rule => !rule.deprecated)
      break
    case 'recommended':
      conditional = conditional.filter(rule => rule.docs?.recommended)
      break
    case 'fixable':
      conditional = conditional.filter(rule => rule.fixable)
      break
    case 'deprecated':
      conditional = conditional.filter(rule => rule.deprecated)
      break
  }

  return conditional
})

const fuse = computed(() => new Fuse(conditionalFiltered.value, {
  keys: ['name', 'docs.description'],
  threshold: 0.5,
}))

const filtered = ref(conditionalFiltered.value)

watchDebounced(
  () => [filters.search, conditionalFiltered.value],
  () => {
    if (!filters.search)
      return filtered.value = conditionalFiltered.value
    filtered.value = fuse.value.search(filters.search).map(i => i.item)
  },
  { debounce: 200 },
)
const isDefaultFilters = computed(() => !(filters.search || filters.plugin || filters.state !== 'using' || filters.status !== 'active'))

function resetFilters() {
  filters.search = ''
  filters.plugin = ''
  filters.state = 'using'
  filters.status = 'active'
}

function escapeCsvCell(value: string | undefined): string {
  const str = value ?? ''
  if (str.includes(',') || str.includes('"') || str.includes('\n'))
    return `"${str.replace(/"/g, '""')}"`
  return str
}

function downloadReport() {
  const headers = ['Rule Name', 'Plugin', 'Level', 'Applied By', 'Fixable', 'Recommended', 'Deprecated', 'Description', 'Docs URL']
  const rows: string[][] = []

  for (const rule of filtered.value) {
    const states = payload.value.ruleToState.get(rule.name)
    if (states?.length) {
      for (const state of states) {
        const config = payload.value.configs[state.configIndex]
        const globs = config?.files?.flat().join(', ')
        const appliedBy = config?.name || (globs ? `anonymous #${state.configIndex + 1} (${globs})` : `anonymous #${state.configIndex + 1}`)
        rows.push([
          rule.name,
          rule.plugin ?? '',
          state.level,
          appliedBy,
          rule.fixable ? 'Yes' : 'No',
          rule.docs?.recommended ? 'Yes' : 'No',
          rule.deprecated ? 'Yes' : 'No',
          rule.docs?.description ?? '',
          rule.docs?.url ?? '',
        ])
      }
    }
    else {
      rows.push([
        rule.name,
        rule.plugin ?? '',
        '',
        '',
        rule.fixable ? 'Yes' : 'No',
        rule.docs?.recommended ? 'Yes' : 'No',
        rule.deprecated ? 'Yes' : 'No',
        rule.docs?.description ?? '',
        rule.docs?.url ?? '',
      ])
    }
  }

  const csv = [headers, ...rows]
    .map(row => row.map(escapeCsvCell).join(','))
    .join('\n')

  const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url
  link.download = 'eslint-rules-report.csv'
  link.click()
  URL.revokeObjectURL(url)
}
</script>

<template>
  <div>
    <div py4 flex="~ col gap-2">
      <div relative flex>
        <input
          v-model="filters.search"
          :class="filters.search ? 'font-mono' : ''"
          placeholder="Search rules..."
          border="~ base rounded-full"
          w-full bg-transparent px3 py2 pl10 outline-none
        >
        <div absolute bottom-0 left-0 top-0 flex="~ items-center justify-center" p4 color-muted>
          <div i-ph-magnifying-glass-duotone />
        </div>
      </div>
      <div grid="~ cols-[max-content_1fr] gap-2" my2 items-center>
        <div text-right text-sm color-muted>
          Plugins
        </div>
        <OptionSelectGroup
          v-model="filters.plugin"
          :options="['', ...pluginNames]"
          :titles="['All', ...pluginNames]"
          :props="[{}, ...pluginNames.map(i => ({
            class: 'font-mono',
            style: filters.plugin === i ? {
              color: getPluginColor(i),
              backgroundColor: getPluginColor(i, 0.1),
            } : {},
          }))]"
        />
        <div text-right text-sm color-muted>
          Usage
        </div>
        <OptionSelectGroup
          v-model="filters.state"
          :options="['', 'using', 'unused', 'error', 'warn', 'off', 'overloads', 'off-only']"
          :titles="['All', 'Using', 'Unused', 'Error', 'Warn', 'Off', 'Overloaded', 'Off Only']"
        >
          <template #default="{ value, title }">
            <div class="flex items-center">
              <div ml--1 mr-1 flex items-center>
                <RuleLevelIcon v-if="value === 'error' || value === 'overloads'" level="error" />
                <RuleLevelIcon v-if="value === 'warn' || value === 'overloads'" level="warn" />
                <RuleLevelIcon v-if="value === 'off' || value === 'off-only' || value === 'overloads'" level="off" />
              </div>
              {{ title || value }}
            </div>
          </template>
        </OptionSelectGroup>
        <div text-right text-sm color-muted>
          State
        </div>
        <OptionSelectGroup
          v-model="filters.status"
          :options="['', 'active', 'recommended', 'fixable', 'deprecated']"
          :titles="['All', 'Active', 'Recommended', 'Fixable', 'Deprecated']"
        >
          <template #default="{ value, title }">
            <div flex items-center gap-1>
              <div v-if="value === 'recommended'" i-ph-check-square-duotone ml--0.5 text-green-700 dark:text-green-300 />
              <div v-if="value === 'fixable'" i-ph-wrench-duotone ml--0.5 text-amber-700 dark:text-amber-300 />
              <div v-if="value === 'deprecated'" i-ph-prohibit-inset-duotone ml--1 color-muted />
              {{ title || value }}
            </div>
          </template>
        </OptionSelectGroup>
      </div>
    </div>

    <div items-center justify-between gap-2 md:flex>
      <div flex="~ gap-2" lt-sm:flex-col>
        <div
          flex="~ inline gap-2 items-center"
          border="~ gray/30 rounded-full" bg-gray:10 px3 py1
        >
          <div i-ph-list-checks-duotone />
          <span>{{ filtered.length }}</span>
          <span color-base>rules {{ isDefaultFilters ? 'enabled' : 'filtered' }}</span>
          <span text-sm color-muted>out of {{ rules.length }} rules</span>
        </div>
        <button
          v-if="!isDefaultFilters"
          flex="~ inline gap-2 items-center self-start"
          border="~ purple-700/30 dark:purple-300/30 rounded-full"
          bg-purple-50 px3 py1 dark:bg-purple-900:20
          @click="resetFilters()"
        >
          <div i-ph-funnel-duotone text-purple-700 dark:text-purple-300 />
          <span color-muted>Clear Filter</span>
          <div
            i-ph-x ml--1 text-sm color-muted hover:color-base
          />
        </button>
      </div>

      <div flex="~ gap-1">
        <button
          v-if="!bpSm"
          btn-action
          :class="{ 'btn-action-active': stateStorage.viewType === 'list' }"
          @click="stateStorage.viewType = 'list'"
        >
          <div i-ph-list-duotone />
          List
        </button>
        <button
          v-if="!bpSm"
          btn-action
          :class="{ 'btn-action-active': stateStorage.viewType === 'grid' }"
          @click="stateStorage.viewType = 'grid'"
        >
          <div i-ph-grid-four-duotone />
          Grid
        </button>
        <button
          btn-action
          title="Download CSV report of filtered rules"
          @click="downloadReport()"
        >
          <div i-ph-download-simple-duotone />
          Export
        </button>
      </div>
    </div>
    <RuleList
      my4
      :rules="filtered"
      :get-bind="(name: string) => ({ class: (payload.ruleToState.get(name)?.length || filters.state === 'unused') ? '' : 'color-muted' })"
    />
  </div>
</template>
