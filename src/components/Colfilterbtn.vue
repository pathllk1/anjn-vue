<script setup lang="ts">
/**
 * ColFilterBtn.vue
 * Reusable column filter button + UPopover for the Bills Reports table.
 * Place at: components/ColFilterBtn.vue
 */

const props = defineProps<{
  /** Column key, e.g. 'bno', 'gtot' */
  col: string
  /** Whether the filter for this col is currently active */
  active: boolean
  /** Whether this popover is open */
  open: boolean
  /** Shared draft state mutated by the parent */
  draft: {
    type: 'set' | 'range'
    values: Set<string>
    min: string
    max: string
  }
  /** Unique values for checkbox list (pre-filtered by filterSearch in parent) */
  uniqueValues: string[]
  /** Current search text inside the popover */
  filterSearch: string
}>()

const emit = defineEmits<{
  /** Request parent to open this col's filter and build draft */
  open: [col: string]
  /** Request parent to close the popover */
  close: []
  'update:filterSearch': [val: string]
  /** Toggle a checkbox value in the draft Set */
  toggleValue: [val: string]
  /** Commit draft → colFilters */
  apply: []
  /** Clear filter for this col */
  clear: []
}>()
</script>

<template>
  <UPopover
    :open="props.open"
    @update:open="(v: boolean) => (v ? emit('open', props.col) : emit('close'))"
  >
    <!-- Trigger: funnel icon button -->
    <button
      type="button"
      class="w-5 h-5 flex items-center justify-center rounded hover:bg-white/20 transition relative"
      :class="props.active ? 'text-amber-300' : 'text-white'"
      :title="`Filter ${props.col}`"
    >
      <UIcon name="i-heroicons-funnel" class="w-3 h-3" />
      <!-- Active indicator dot -->
      <span
        v-if="props.active"
        class="absolute -top-1 -right-1 w-2 h-2 bg-amber-400 rounded-full"
      />
    </button>

    <!-- Popover content -->
    <template #content>
      <div class="w-64 p-3 space-y-3 text-sm">

        <!-- Header -->
        <div class="flex items-center justify-between">
          <span class="text-xs font-bold text-gray-600 uppercase tracking-wide">
            Filter
          </span>
          <button
            class="text-gray-400 hover:text-gray-600 transition"
            @click="emit('close')"
          >
            <UIcon name="i-heroicons-x-mark" class="w-4 h-4" />
          </button>
        </div>

        <!-- ── Number range ──────────────────────────────────────────────── -->
        <template v-if="props.draft.type === 'range'">
          <p class="text-[10px] text-gray-400 uppercase tracking-wide">Filter by range</p>
          <div class="flex gap-2">
            <div class="flex-1 min-w-0">
              <label class="block text-[10px] text-gray-500 mb-0.5">Min</label>
              <input
                type="number"
                :value="props.draft.min"
                placeholder="0"
                class="w-full px-2 py-1.5 text-xs border border-gray-300 rounded-lg bg-white
                       focus:outline-none focus:ring-2 focus:ring-emerald-400 focus:border-emerald-400 transition"
                @input="props.draft.min = ($event.target as HTMLInputElement).value"
              >
            </div>
            <div class="flex-1 min-w-0">
              <label class="block text-[10px] text-gray-500 mb-0.5">Max</label>
              <input
                type="number"
                :value="props.draft.max"
                placeholder="Any"
                class="w-full px-2 py-1.5 text-xs border border-gray-300 rounded-lg bg-white
                       focus:outline-none focus:ring-2 focus:ring-emerald-400 focus:border-emerald-400 transition"
                @input="props.draft.max = ($event.target as HTMLInputElement).value"
              >
            </div>
          </div>
        </template>

        <!-- ── Checkbox list ─────────────────────────────────────────────── -->
        <template v-else>
          <!-- Inner search -->
          <UInput
            :model-value="props.filterSearch"
            icon="i-heroicons-magnifying-glass"
            placeholder="Search values…"
            size="xs"
            @update:model-value="emit('update:filterSearch', $event)"
          />

          <!-- Checkbox list -->
          <div class="max-h-48 overflow-y-auto space-y-0.5 pr-0.5">
            <label
              v-for="val in props.uniqueValues"
              :key="val"
              class="flex items-center gap-2 px-1.5 py-1 rounded hover:bg-gray-50 cursor-pointer"
            >
              <input
                type="checkbox"
                :checked="props.draft.values.has(val)"
                class="rounded border-gray-300 text-orange-500 focus:ring-orange-400 flex-shrink-0"
                @change="emit('toggleValue', val)"
              >
              <span class="text-xs text-gray-700 truncate">{{ val }}</span>
            </label>
            <p
              v-if="!props.uniqueValues.length"
              class="text-center text-xs text-gray-400 py-3"
            >
              No values found
            </p>
          </div>
        </template>

        <!-- Footer actions -->
        <div class="flex gap-2 pt-1 border-t border-gray-100">
          <UButton
            label="Apply"
            color="primary"
            size="xs"
            class="flex-1"
            @click="emit('apply')"
          />
          <UButton
            label="Clear"
            color="neutral"
            variant="outline"
            size="xs"
            @click="emit('clear')"
          />
        </div>

      </div>
    </template>
  </UPopover>
</template>