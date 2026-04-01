<script setup lang="ts">
const props = defineProps<{
  col: string
  active: boolean
  open: boolean
  draft: { type: 'set' | 'range'; values: Set<string>; min: string; max: string }
  uniqueValues: string[]
  filterSearch: string
}>()
const emit = defineEmits<{
  open:                  [col: string]
  close:                 []
  'update:filterSearch': [val: string]
  toggleValue:           [val: string]
  apply:                 []
  clear:                 []
}>()
</script>

<template>
  <UPopover :open="props.open" @update:open="(v) => v ? emit('open', props.col) : emit('close')">
    <button
      type="button"
      class="w-5 h-5 flex items-center justify-center rounded hover:bg-white/20 transition relative"
      :class="props.active ? 'text-amber-300' : 'text-white'"
    >
      <UIcon name="i-heroicons-funnel" class="w-3 h-3" />
      <span v-if="props.active" class="absolute -top-1 -right-1 w-2 h-2 bg-amber-400 rounded-full" />
    </button>
    <template #content>
      <div class="w-64 p-3 space-y-3 bg-white rounded-xl shadow-xl border border-gray-200 text-sm">
        <div class="flex items-center justify-between">
          <span class="text-xs font-bold text-gray-600 uppercase tracking-wide">Filter</span>
          <button class="text-gray-400 hover:text-gray-600" @click="emit('close')">
            <UIcon name="i-heroicons-x-mark" class="w-4 h-4" />
          </button>
        </div>

        <!-- Number range -->
        <template v-if="props.draft.type === 'range'">
          <div class="flex gap-2">
            <div class="flex-1">
              <label class="block text-[10px] text-gray-500 mb-1">Min</label>
              <input type="number" :value="props.draft.min" @input="props.draft.min = ($event.target as HTMLInputElement).value"
                class="w-full px-2 py-1.5 text-xs border border-gray-300 rounded-lg focus:ring-2 focus:ring-emerald-400 focus:border-emerald-400"
                placeholder="0"
              >
            </div>
            <div class="flex-1">
              <label class="block text-[10px] text-gray-500 mb-1">Max</label>
              <input type="number" :value="props.draft.max" @input="props.draft.max = ($event.target as HTMLInputElement).value"
                class="w-full px-2 py-1.5 text-xs border border-gray-300 rounded-lg focus:ring-2 focus:ring-emerald-400 focus:border-emerald-400"
                placeholder="Any"
              >
            </div>
          </div>
        </template>

        <!-- Checkbox list -->
        <template v-else>
          <UInput :model-value="props.filterSearch" @update:model-value="emit('update:filterSearch', $event)"
            icon="i-heroicons-magnifying-glass" placeholder="Search…" size="xs" />
          <div class="max-h-48 overflow-y-auto space-y-1 pr-1">
            <label v-for="val in props.uniqueValues" :key="val"
              class="flex items-center gap-2 px-1.5 py-1 rounded hover:bg-gray-50 cursor-pointer text-xs">
              <input type="checkbox"
                :checked="props.draft.values.has(val)"
                @change="emit('toggleValue', val)"
                class="rounded border-gray-300 text-orange-500 focus:ring-orange-400"
              >
              <span class="truncate text-gray-700">{{ val }}</span>
            </label>
            <p v-if="!props.uniqueValues.length" class="text-center text-gray-400 py-2">No values</p>
          </div>
        </template>

        <div class="flex gap-2 pt-1 border-t border-gray-100">
          <UButton label="Apply" color="primary" size="xs" class="flex-1" @click="emit('apply')" />
          <UButton label="Clear"  color="neutral" variant="outline" size="xs" @click="emit('clear')" />
        </div>
      </div>
    </template>
  </UPopover>
</template>
