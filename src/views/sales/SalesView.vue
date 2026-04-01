<script setup lang="ts">
import { ref, computed, watch, onMounted, onUnmounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { api } from '@/utils/api'
import ColFilterBtn from '@/components/ColFilterBtn.vue'
import ColFilterPanel from '@/components/ColFilterPanel.vue'

// ─── Route context ────────────────────────────────────────────────────────────
const route  = useRoute()
const router = useRouter()

const isReturnAction = computed(() => route.query.action === 'return')
const returnTypeHint = computed(() =>
  route.query.type === 'PURCHASE' ? 'Purchase (Debit Note)' : 'Sales (Credit Note)'
)

// ─── Bills state ──────────────────────────────────────────────────────────────
const allBills   = ref<any[]>([])
const loading    = ref(false)
const fetchError = ref('')

// ─── Global filters ───────────────────────────────────────────────────────────
const search     = ref('')
const typeFilter = ref('')
const dateFrom   = ref('')
const dateTo     = ref('')

// ─── Sort ─────────────────────────────────────────────────────────────────────
const sortCol = ref('')
const sortDir = ref<'asc' | 'desc'>('asc')

// ─── Pagination ───────────────────────────────────────────────────────────────
const currentPage   = ref(1)
const itemsPerPage  = ref(10)

// ─── Column filters (Excel-style) ─────────────────────────────────────────────
type RangeFilter = { min: string; max: string }
type ColFilter   = Set<string> | RangeFilter | null

const colFilters = ref<Record<string, ColFilter>>({
  bno: null, bdate: null, firm: null, btype: null,
  gtot: null, tax: null, ntot: null, status: null,
})
const openFilterCol = ref<string | null>(null)
const filterDraft   = ref<{ type: 'set' | 'range'; values: Set<string>; min: string; max: string }>({
  type: 'set', values: new Set(), min: '', max: '',
})
// temp search inside popover checkbox list
const filterSearch = ref('')

// ─── Bill detail modal ────────────────────────────────────────────────────────
const modalOpen     = ref(false)
const detailBill    = ref<any>(null)
const detailLoading = ref(false)

// ─── Cancel state ─────────────────────────────────────────────────────────────
const cancelFormVisible = ref(false)
const cancelReason      = ref('')
const cancelRemarks     = ref('')
const cancelling        = ref(false)

// ─── Select options ───────────────────────────────────────────────────────────
const typeOptions = [
  { label: 'Sales',                         value: 'SALES' },
  { label: 'Purchase',                      value: 'PURCHASE' },
  { label: 'Credit Note (Sales Return)',    value: 'CREDIT_NOTE' },
  { label: 'Debit Note (Purchase Return)', value: 'DEBIT_NOTE' },
]

const itemsPerPageOptions = [
  { label: '10 / page',  value: 10 },
  { label: '25 / page',  value: 25 },
  { label: '50 / page',  value: 50 },
  { label: '100 / page', value: 100 },
]

const cancelReasonOptions = [
  { label: 'Data entry error',     value: 'DATA_ENTRY_ERROR' },
  { label: 'Order cancelled',      value: 'ORDER_CANCELLED' },
  { label: 'Duplicate bill',       value: 'DUPLICATE' },
  { label: 'Party request',        value: 'PARTY_REQUEST' },
  { label: 'Other',                value: 'OTHER' },
]

// ─── Helpers ──────────────────────────────────────────────────────────────────
function formatDate(d: string) {
  if (!d) return '—'
  return new Date(d).toLocaleDateString('en-IN', { day: '2-digit', month: '2-digit', year: 'numeric' })
}

function fmtAmt(n: number) {
  return '₹' + (n || 0).toFixed(2)
}

function getBillPartyName(bill: any) {
  return bill?.supply || '—'
}

function getTax(bill: any) {
  return (bill.cgst || 0) + (bill.sgst || 0) + (bill.igst || 0)
}

function isCancelled(bill: any) {
  return (bill?.status || 'ACTIVE') === 'CANCELLED'
}

function apiSegment(btype: string) {
  return (btype || 'SALES').toUpperCase() === 'PURCHASE' ? 'purchase' : 'sales'
}

// ─── Column config ────────────────────────────────────────────────────────────
const COL_CONFIG: Record<string, { label: string; type: 'text' | 'choice' | 'number'; getValue: (b: any) => any }> = {
  bno:    { label: 'Bill No',  type: 'text',   getValue: b => b.bno || '' },
  bdate:  { label: 'Date',     type: 'text',   getValue: b => formatDate(b.bdate) },
  firm:   { label: 'Party',    type: 'text',   getValue: b => getBillPartyName(b) },
  btype:  { label: 'Type',     type: 'choice', getValue: b => b.btype || 'SALES' },
  gtot:   { label: 'Taxable',  type: 'number', getValue: b => b.gtot || 0 },
  tax:    { label: 'Tax',      type: 'number', getValue: b => getTax(b) },
  ntot:   { label: 'Total',    type: 'number', getValue: b => b.ntot || 0 },
  status: { label: 'Status',   type: 'choice', getValue: b => b.status || 'ACTIVE' },
}

function isColFilterActive(col: string) {
  const f = colFilters.value[col]
  if (!f) return false
  if (f instanceof Set) return f.size > 0
  return (f as RangeFilter).min !== '' || (f as RangeFilter).max !== ''
}

function matchesColFilters(bill: any) {
  for (const [col, state] of Object.entries(colFilters.value)) {
    if (!state) continue
    const cfg = COL_CONFIG[col]
    if (!cfg) continue
    const val = cfg.getValue(bill)
    if (cfg.type === 'number') {
      const { min, max } = state as RangeFilter
      const n = Number(val)
      if (min !== '' && !isNaN(Number(min)) && n < Number(min)) return false
      if (max !== '' && !isNaN(Number(max)) && n > Number(max)) return false
    } else {
      const s = state as Set<string>
      if (s.size > 0 && !s.has(String(val))) return false
    }
  }
  return true
}

// ─── Computed pipelines ───────────────────────────────────────────────────────
const filteredBills = computed(() => {
  let bills = allBills.value
  const q = search.value.trim().toLowerCase()
  if (q) {
    bills = bills.filter(b =>
      (b.bno || '').toLowerCase().includes(q) ||
      (b.supply || '').toLowerCase().includes(q) ||
      (b.btype || '').toLowerCase().includes(q)
    )
  }
  if (typeFilter.value) {
    bills = bills.filter(b => (b.btype || 'SALES').toUpperCase() === typeFilter.value)
  }
  if (dateFrom.value) {
    const from = new Date(dateFrom.value)
    bills = bills.filter(b => b.bdate && new Date(b.bdate) >= from)
  }
  if (dateTo.value) {
    const to = new Date(dateTo.value)
    to.setHours(23, 59, 59, 999)
    bills = bills.filter(b => b.bdate && new Date(b.bdate) <= to)
  }
  return bills.filter(b => matchesColFilters(b))
})

const sortedBills = computed(() => {
  if (!sortCol.value) return filteredBills.value
  return [...filteredBills.value].sort((a, b) => {
    const cfg = COL_CONFIG[sortCol.value]
    const va  = cfg ? cfg.getValue(a) : a[sortCol.value] ?? ''
    const vb  = cfg ? cfg.getValue(b) : b[sortCol.value] ?? ''
    let cmp   = typeof va === 'number' && typeof vb === 'number'
      ? va - vb
      : String(va).localeCompare(String(vb))
    return sortDir.value === 'asc' ? cmp : -cmp
  })
})

const paginatedBills = computed(() => {
  const start = (currentPage.value - 1) * itemsPerPage.value
  return sortedBills.value.slice(start, start + itemsPerPage.value)
})

const totalPages = computed(() => Math.max(1, Math.ceil(sortedBills.value.length / itemsPerPage.value)))

const summary = computed(() => {
  const active = allBills.value.filter(b => !isCancelled(b))
  return {
    totalBills:  allBills.value.length,
    revenue:     active.reduce((s, b) => s + (b.gtot || 0), 0),
    totalTax:    active.reduce((s, b) => s + getTax(b), 0),
    activeBills: active.length,
  }
})

// unique values for the checkbox list in col filter popovers
const colUniqueValues = computed(() => {
  const col = openFilterCol.value
  if (!col) return []
  const cfg = COL_CONFIG[col]
  if (!cfg) return []
  const q   = filterSearch.value.trim().toLowerCase()
  const all = [...new Set(allBills.value.map(b => String(cfg.getValue(b))))].sort()
  return q ? all.filter(v => v.toLowerCase().includes(q)) : all
})

// ─── Data loading ─────────────────────────────────────────────────────────────
async function loadBills() {
  loading.value  = true
  fetchError.value = ''
  try {
    const data = await api.get('/inventory/sales/bills')
    let extracted: any[] = []
    if (Array.isArray(data)) {
      extracted = data
    } else {
      for (const t of [data.data, data.data?.rows, data.data?.bills, data.bills, data.rows]) {
        if (Array.isArray(t)) { extracted = t; break }
      }
    }
    allBills.value    = extracted
    currentPage.value = 1
  } catch (err: any) {
    fetchError.value = err.message || 'Failed to load bills'
  } finally {
    loading.value = false
  }
}

// ─── Sort ─────────────────────────────────────────────────────────────────────
function toggleSort(col: string) {
  if (sortCol.value === col) sortDir.value = sortDir.value === 'asc' ? 'desc' : 'asc'
  else { sortCol.value = col; sortDir.value = 'asc' }
  currentPage.value = 1
}

// ─── Column filter ────────────────────────────────────────────────────────────
function openColFilter(col: string) {
  const cfg = COL_CONFIG[col]
  if (!cfg) return
  filterSearch.value = ''
  const existing = colFilters.value[col]
  if (cfg.type === 'number') {
    const f = existing as RangeFilter | null
    filterDraft.value = { type: 'range', values: new Set(), min: f?.min ?? '', max: f?.max ?? '' }
  } else {
    const f = existing as Set<string> | null
    filterDraft.value = { type: 'set', values: f ? new Set(f) : new Set(), min: '', max: '' }
  }
  openFilterCol.value = col
}

function toggleDraftValue(val: string) {
  if (filterDraft.value.values.has(val)) filterDraft.value.values.delete(val)
  else filterDraft.value.values.add(val)
  filterDraft.value = { ...filterDraft.value }  // trigger reactivity
}

function applyDraft() {
  const col = openFilterCol.value
  if (!col) return
  if (filterDraft.value.type === 'range') {
    const { min, max } = filterDraft.value
    colFilters.value[col] = (min === '' && max === '') ? null : { min, max }
  } else {
    colFilters.value[col] = filterDraft.value.values.size > 0 ? new Set(filterDraft.value.values) : null
  }
  openFilterCol.value = null
  currentPage.value   = 1
}

function clearDraft() {
  colFilters.value[openFilterCol.value!] = null
  openFilterCol.value = null
  currentPage.value   = 1
}

// ─── Bill detail modal ────────────────────────────────────────────────────────
async function showBillDetails(billId: string) {
  modalOpen.value        = true
  detailBill.value       = null
  detailLoading.value    = true
  cancelFormVisible.value = false
  cancelReason.value     = ''
  cancelRemarks.value    = ''

  try {
    const billMeta = allBills.value.find(b => b._id === billId || b.id === billId)
    const seg      = apiSegment(billMeta?.btype || 'SALES')
    const data     = await api.get(`/inventory/${seg}/bills/${billId}`)
    detailBill.value = data.success ? data.data : data
  } catch (err: any) {
    modalOpen.value = false
    alert('Failed to load bill details: ' + err.message)
  } finally {
    detailLoading.value = false
  }
}

// ─── Cancel bill ──────────────────────────────────────────────────────────────
async function cancelBill() {
  if (!cancelReason.value) { alert('Please select a cancellation reason.'); return }
  if (!confirm('Are you sure you want to cancel this bill? This cannot be undone.')) return

  cancelling.value = true
  try {
    const bill = detailBill.value
    const seg  = apiSegment(bill.btype)
    await api.put(`/inventory/${seg}/bills/${bill._id}/cancel`, {
      reason:  cancelReason.value,
      remarks: cancelRemarks.value.trim(),
    })
    detailBill.value    = { ...bill, status: 'CANCELLED' }
    cancelFormVisible.value = false
    cancelReason.value  = ''
    cancelRemarks.value = ''
    await loadBills()
  } catch (err: any) {
    alert('Failed to cancel bill: ' + err.message)
  } finally {
    cancelling.value = false
  }
}

// ─── Download helpers ──────────────────────────────────────────────────────────
function triggerDownload(url: string, filename: string) {
  const token = typeof localStorage !== 'undefined' ? localStorage.getItem('token') : null
  fetch(url, {
    credentials: 'include',
    headers: token ? { Authorization: `Bearer ${token}` } : {},
  })
    .then(r => { if (!r.ok) throw new Error('Download failed'); return r.blob() })
    .then(blob => {
      const a     = document.createElement('a')
      a.href      = URL.createObjectURL(blob)
      a.download  = filename
      document.body.appendChild(a); a.click()
      document.body.removeChild(a); URL.revokeObjectURL(a.href)
    })
    .catch(e => alert(e.message))
}

function downloadBillPdf(bill: any) {
  const seg = apiSegment(bill.btype)
  triggerDownload(`/api/inventory/${seg}/bills/${bill._id}/pdf`, `Invoice_${bill._id}.pdf`)
}

function downloadBillExcel(bill: any) {
  const seg = apiSegment(bill.btype)
  triggerDownload(`/api/inventory/${seg}/bills/${bill._id}/excel`, `Invoice_${bill._id}.xlsx`)
}

// ─── Bulk export ───────────────────────────────────────────────────────────────
function buildExportParams() {
  const p = new URLSearchParams()
  if (typeFilter.value) p.append('type',       typeFilter.value)
  if (search.value)     p.append('searchTerm', search.value)
  if (dateFrom.value)   p.append('dateFrom',   dateFrom.value)
  if (dateTo.value)     p.append('dateTo',     dateTo.value)
  return p.toString()
}

function exportToExcel() {
  triggerDownload(
    `/api/inventory/sales/bills/export?${buildExportParams()}`,
    `bills-${new Date().toISOString().split('T')[0]}.xlsx`
  )
}

function exportToPdf() {
  triggerDownload(
    `/api/inventory/sales/bills/export/pdf?${buildExportParams()}`,
    `bills-report-${new Date().toISOString().split('T')[0]}.pdf`
  )
}

// ─── Edit / Return navigation ─────────────────────────────────────────────────
function editBill(bill: any) {
  sessionStorage.setItem('editBillId', bill._id)
  router.push('/sales/create')
}

function initiateReturn(bill: any) {
  sessionStorage.setItem('returnFromBillId', bill._id)
  router.push('/sales/create')
}

// ─── Keyboard shortcuts ───────────────────────────────────────────────────────
function handleKeydown(e: KeyboardEvent) {
  if (openFilterCol.value && e.key === 'Escape') { openFilterCol.value = null; return }
  if (modalOpen.value      && e.key === 'Escape') { modalOpen.value = false; return }
  const tag = (document.activeElement as HTMLElement)?.tagName?.toLowerCase()
  if (tag === 'input' || tag === 'textarea' || tag === 'select') return
  if ((e.ctrlKey || e.metaKey) && e.key === 'f') {
    e.preventDefault()
    document.getElementById('search-bills')?.focus()
    return
  }
  if (e.key === 'r' || e.key === 'R') { loadBills(); return }
  if (e.key === 'e' || e.key === 'E') { exportToExcel(); return }
  if (e.key === 'p' || e.key === 'P') { exportToPdf();   return }
  if (e.key === 'ArrowLeft'  && currentPage.value > 1)             currentPage.value--
  if (e.key === 'ArrowRight' && currentPage.value < totalPages.value) currentPage.value++
}

watch([search, typeFilter, dateFrom, dateTo], () => { currentPage.value = 1 })

onMounted(() => {
  loadBills()
  document.addEventListener('keydown', handleKeydown)
})
onUnmounted(() => {
  document.removeEventListener('keydown', handleKeydown)
})
</script>

<template>
  <div class="px-3 py-3 space-y-3" @click.self="openFilterCol = null">

    <!-- ── Header ────────────────────────────────────────────────────────────── -->
    <div class="flex flex-col sm:flex-row sm:items-center sm:justify-between gap-2">
      <div class="flex items-center gap-3">
        <div class="w-9 h-9 bg-gradient-to-br from-orange-500 to-amber-600 rounded-lg flex items-center justify-center flex-shrink-0">
          <UIcon name="i-heroicons-chart-bar" class="w-5 h-5 text-white" />
        </div>
        <div>
          <h1 class="text-xl font-bold text-gray-900 leading-tight">Bills Reports</h1>
          <p class="text-xs text-gray-500">View and analyze all bills and transactions</p>
        </div>
      </div>
      <UButton
        to="/"
        icon="i-heroicons-arrow-left"
        label="Home"
        color="warning"
        variant="subtle"
        size="xs"
        class="self-start sm:self-auto"
      />
    </div>

    <!-- ── Return action notice ───────────────────────────────────────────────── -->
    <UAlert
      v-if="isReturnAction"
      icon="i-heroicons-exclamation-triangle"
      :title="`Create ${returnTypeHint} Return`"
      :description="`Please locate the original bill below and click the Return button to proceed.`"
      color="warning"
      variant="subtle"
    />

    <!-- ── Error ──────────────────────────────────────────────────────────────── -->
    <UAlert
      v-if="fetchError"
      icon="i-heroicons-exclamation-circle"
      :title="fetchError"
      color="error"
      variant="subtle"
    />

    <!-- ── Summary cards ──────────────────────────────────────────────────────── -->
    <div class="grid grid-cols-2 lg:grid-cols-4 gap-2">
      <UCard :ui="{ body: 'p-3' }">
        <div class="flex items-center gap-3">
          <div class="w-8 h-8 rounded-lg bg-blue-100 flex items-center justify-center flex-shrink-0">
            <UIcon name="i-heroicons-document-text" class="w-4 h-4 text-blue-600" />
          </div>
          <div class="min-w-0">
            <p class="text-xs text-gray-500 truncate">Total Bills</p>
            <p class="text-lg font-bold text-gray-900 leading-tight">{{ summary.totalBills }}</p>
          </div>
        </div>
      </UCard>

      <UCard :ui="{ body: 'p-3' }">
        <div class="flex items-center gap-3">
          <div class="w-8 h-8 rounded-lg bg-green-100 flex items-center justify-center flex-shrink-0">
            <UIcon name="i-heroicons-currency-rupee" class="w-4 h-4 text-green-600" />
          </div>
          <div class="min-w-0">
            <p class="text-xs text-gray-500 truncate">Revenue</p>
            <p class="text-lg font-bold text-gray-900 leading-tight">{{ fmtAmt(summary.revenue) }}</p>
          </div>
        </div>
      </UCard>

      <UCard :ui="{ body: 'p-3' }">
        <div class="flex items-center gap-3">
          <div class="w-8 h-8 rounded-lg bg-orange-100 flex items-center justify-center flex-shrink-0">
            <UIcon name="i-heroicons-receipt-percent" class="w-4 h-4 text-orange-600" />
          </div>
          <div class="min-w-0">
            <p class="text-xs text-gray-500 truncate">Total Tax</p>
            <p class="text-lg font-bold text-gray-900 leading-tight">{{ fmtAmt(summary.totalTax) }}</p>
          </div>
        </div>
      </UCard>

      <UCard :ui="{ body: 'p-3' }">
        <div class="flex items-center gap-3">
          <div class="w-8 h-8 rounded-lg bg-purple-100 flex items-center justify-center flex-shrink-0">
            <UIcon name="i-heroicons-check-circle" class="w-4 h-4 text-purple-600" />
          </div>
          <div class="min-w-0">
            <p class="text-xs text-gray-500 truncate">Active Bills</p>
            <p class="text-lg font-bold text-gray-900 leading-tight">{{ summary.activeBills }}</p>
          </div>
        </div>
      </UCard>
    </div>

    <!-- ── Controls ───────────────────────────────────────────────────────────── -->
    <UCard :ui="{ body: 'p-3' }">
      <div class="flex flex-col gap-2">
        <!-- Search + Type filter -->
        <div class="flex flex-col sm:flex-row gap-2">
          <UInput
            id="search-bills"
            v-model="search"
            placeholder="Search bills… (Ctrl+F)"
            icon="i-heroicons-magnifying-glass"
            class="flex-1"
            size="sm"
          />
          <USelect
            v-model="typeFilter"
            :items="typeOptions"
            placeholder="All types"
            size="sm"
            class="sm:w-56"
          />
        </div>

        <!-- Date range + action buttons -->
        <div class="flex flex-col sm:flex-row items-start sm:items-center gap-2 justify-between">
          <div class="flex flex-wrap gap-2 items-center">
            <div class="flex items-center gap-1.5">
              <label class="text-xs text-gray-500 whitespace-nowrap">From</label>
              <input
                v-model="dateFrom"
                type="date"
                class="px-2 py-1.5 text-sm border border-gray-300 rounded-md focus:ring-2 focus:ring-orange-400 focus:border-orange-400 bg-gray-50"
              >
            </div>
            <div class="flex items-center gap-1.5">
              <label class="text-xs text-gray-500 whitespace-nowrap">To</label>
              <input
                v-model="dateTo"
                type="date"
                class="px-2 py-1.5 text-sm border border-gray-300 rounded-md focus:ring-2 focus:ring-orange-400 focus:border-orange-400 bg-gray-50"
              >
            </div>
          </div>

          <div class="flex items-center gap-2">
            <UButton
              icon="i-heroicons-arrow-path"
              label="Refresh"
              color="info"
              size="xs"
              :loading="loading"
              @click="loadBills"
            >
              <template #trailing>
                <kbd class="px-1 py-0.5 bg-blue-500 border border-blue-400 rounded font-mono text-blue-100 text-xs leading-none">R</kbd>
              </template>
            </UButton>
            <UButton
              icon="i-heroicons-arrow-down-tray"
              label="Export PDF"
              color="error"
              size="xs"
              @click="exportToPdf"
            >
              <template #trailing>
                <kbd class="px-1 py-0.5 bg-red-500 border border-red-400 rounded font-mono text-red-100 text-xs leading-none">P</kbd>
              </template>
            </UButton>
            <UButton
              icon="i-heroicons-arrow-down-tray"
              label="Export"
              color="success"
              size="xs"
              @click="exportToExcel"
            >
              <template #trailing>
                <kbd class="px-1 py-0.5 bg-green-500 border border-green-400 rounded font-mono text-green-100 text-xs leading-none">E</kbd>
              </template>
            </UButton>
          </div>
        </div>
      </div>
    </UCard>

    <!-- ── Bills Table ────────────────────────────────────────────────────────── -->
    <div class="bg-white rounded-lg border border-gray-200 overflow-hidden">
      <div class="overflow-x-auto">
        <table class="min-w-full divide-y divide-gray-200 text-sm">
          <thead class="bg-gradient-to-r from-emerald-600 via-cyan-600 to-violet-600">
            <tr>
              <!-- Bill No -->
              <th class="px-3 py-2.5 text-left text-xs font-semibold text-white uppercase tracking-wider">
                <div class="flex items-center justify-between gap-1">
                  <button class="flex items-center gap-1 hover:opacity-80 transition" @click="toggleSort('bno')">
                    Bill No
                    <UIcon
                      :name="sortCol === 'bno' ? (sortDir === 'asc' ? 'i-heroicons-chevron-up' : 'i-heroicons-chevron-down') : 'i-heroicons-chevron-up-down'"
                      class="w-3 h-3"
                      :class="sortCol === 'bno' ? 'opacity-100' : 'opacity-40'"
                    />
                  </button>
                  <!-- Col filter button -->
                  <UPopover :open="openFilterCol === 'bno'" @update:open="(v) => { if(v) openColFilter('bno'); else openFilterCol = null }">
                    <button
                      type="button"
                      class="w-5 h-5 flex items-center justify-center rounded hover:bg-white/20 transition relative"
                      :class="isColFilterActive('bno') ? 'text-amber-300' : 'text-white'"
                    >
                      <UIcon name="i-heroicons-funnel" class="w-3 h-3" />
                      <span v-if="isColFilterActive('bno')" class="absolute -top-1 -right-1 w-2 h-2 bg-amber-400 rounded-full" />
                    </button>
                    <template #content>
                      <ColFilterPanel
                        col="bno"
                        :draft="filterDraft"
                        :unique-values="colUniqueValues"
                        :filter-search="filterSearch"
                        @update:filter-search="filterSearch = $event"
                        @toggle-value="toggleDraftValue"
                        @apply="applyDraft"
                        @clear="clearDraft"
                        @close="openFilterCol = null"
                      />
                    </template>
                  </UPopover>
                </div>
              </th>

              <!-- Date -->
              <th class="px-3 py-2.5 text-left text-xs font-semibold text-white uppercase tracking-wider">
                <div class="flex items-center justify-between gap-1">
                  <button class="flex items-center gap-1 hover:opacity-80 transition" @click="toggleSort('bdate')">
                    Date
                    <UIcon
                      :name="sortCol === 'bdate' ? (sortDir === 'asc' ? 'i-heroicons-chevron-up' : 'i-heroicons-chevron-down') : 'i-heroicons-chevron-up-down'"
                      class="w-3 h-3"
                      :class="sortCol === 'bdate' ? 'opacity-100' : 'opacity-40'"
                    />
                  </button>
                  <ColFilterBtn col="bdate" :active="isColFilterActive('bdate')" :open="openFilterCol === 'bdate'"
                    @open="openColFilter('bdate')" @close="openFilterCol = null"
                    :draft="filterDraft" :unique-values="colUniqueValues" :filter-search="filterSearch"
                    @update:filter-search="filterSearch = $event" @toggle-value="toggleDraftValue"
                    @apply="applyDraft" @clear="clearDraft"
                  />
                </div>
              </th>

              <!-- Party (hidden on mobile) -->
              <th class="px-3 py-2.5 text-left text-xs font-semibold text-white uppercase tracking-wider hidden sm:table-cell">
                <div class="flex items-center justify-between gap-1">
                  <button class="flex items-center gap-1 hover:opacity-80 transition" @click="toggleSort('firm')">
                    Party
                    <UIcon
                      :name="sortCol === 'firm' ? (sortDir === 'asc' ? 'i-heroicons-chevron-up' : 'i-heroicons-chevron-down') : 'i-heroicons-chevron-up-down'"
                      class="w-3 h-3"
                      :class="sortCol === 'firm' ? 'opacity-100' : 'opacity-40'"
                    />
                  </button>
                  <ColFilterBtn col="firm" :active="isColFilterActive('firm')" :open="openFilterCol === 'firm'"
                    @open="openColFilter('firm')" @close="openFilterCol = null"
                    :draft="filterDraft" :unique-values="colUniqueValues" :filter-search="filterSearch"
                    @update:filter-search="filterSearch = $event" @toggle-value="toggleDraftValue"
                    @apply="applyDraft" @clear="clearDraft"
                  />
                </div>
              </th>

              <!-- Type -->
              <th class="px-3 py-2.5 text-left text-xs font-semibold text-white uppercase tracking-wider">
                <div class="flex items-center justify-between gap-1">
                  <button class="flex items-center gap-1 hover:opacity-80 transition" @click="toggleSort('btype')">
                    Type
                    <UIcon
                      :name="sortCol === 'btype' ? (sortDir === 'asc' ? 'i-heroicons-chevron-up' : 'i-heroicons-chevron-down') : 'i-heroicons-chevron-up-down'"
                      class="w-3 h-3"
                      :class="sortCol === 'btype' ? 'opacity-100' : 'opacity-40'"
                    />
                  </button>
                  <ColFilterBtn col="btype" :active="isColFilterActive('btype')" :open="openFilterCol === 'btype'"
                    @open="openColFilter('btype')" @close="openFilterCol = null"
                    :draft="filterDraft" :unique-values="colUniqueValues" :filter-search="filterSearch"
                    @update:filter-search="filterSearch = $event" @toggle-value="toggleDraftValue"
                    @apply="applyDraft" @clear="clearDraft"
                  />
                </div>
              </th>

              <!-- Taxable (hidden on mobile) -->
              <th class="px-3 py-2.5 text-right text-xs font-semibold text-white uppercase tracking-wider hidden md:table-cell">
                <div class="flex items-center justify-end gap-1">
                  <ColFilterBtn col="gtot" :active="isColFilterActive('gtot')" :open="openFilterCol === 'gtot'"
                    @open="openColFilter('gtot')" @close="openFilterCol = null"
                    :draft="filterDraft" :unique-values="colUniqueValues" :filter-search="filterSearch"
                    @update:filter-search="filterSearch = $event" @toggle-value="toggleDraftValue"
                    @apply="applyDraft" @clear="clearDraft"
                  />
                  <button class="flex items-center gap-1 hover:opacity-80 transition" @click="toggleSort('gtot')">
                    Taxable
                    <UIcon
                      :name="sortCol === 'gtot' ? (sortDir === 'asc' ? 'i-heroicons-chevron-up' : 'i-heroicons-chevron-down') : 'i-heroicons-chevron-up-down'"
                      class="w-3 h-3"
                      :class="sortCol === 'gtot' ? 'opacity-100' : 'opacity-40'"
                    />
                  </button>
                </div>
              </th>

              <!-- Tax (hidden on mobile) -->
              <th class="px-3 py-2.5 text-right text-xs font-semibold text-white uppercase tracking-wider hidden md:table-cell">
                <div class="flex items-center justify-end gap-1">
                  <ColFilterBtn col="tax" :active="isColFilterActive('tax')" :open="openFilterCol === 'tax'"
                    @open="openColFilter('tax')" @close="openFilterCol = null"
                    :draft="filterDraft" :unique-values="colUniqueValues" :filter-search="filterSearch"
                    @update:filter-search="filterSearch = $event" @toggle-value="toggleDraftValue"
                    @apply="applyDraft" @clear="clearDraft"
                  />
                  <button class="flex items-center gap-1 hover:opacity-80 transition" @click="toggleSort('tax')">
                    Tax
                    <UIcon
                      :name="sortCol === 'tax' ? (sortDir === 'asc' ? 'i-heroicons-chevron-up' : 'i-heroicons-chevron-down') : 'i-heroicons-chevron-up-down'"
                      class="w-3 h-3"
                      :class="sortCol === 'tax' ? 'opacity-100' : 'opacity-40'"
                    />
                  </button>
                </div>
              </th>

              <!-- Total -->
              <th class="px-3 py-2.5 text-right text-xs font-semibold text-white uppercase tracking-wider">
                <div class="flex items-center justify-end gap-1">
                  <ColFilterBtn col="ntot" :active="isColFilterActive('ntot')" :open="openFilterCol === 'ntot'"
                    @open="openColFilter('ntot')" @close="openFilterCol = null"
                    :draft="filterDraft" :unique-values="colUniqueValues" :filter-search="filterSearch"
                    @update:filter-search="filterSearch = $event" @toggle-value="toggleDraftValue"
                    @apply="applyDraft" @clear="clearDraft"
                  />
                  Total
                </div>
              </th>

              <!-- Status -->
              <th class="px-3 py-2.5 text-left text-xs font-semibold text-white uppercase tracking-wider">
                <div class="flex items-center justify-between gap-1">
                  Status
                  <ColFilterBtn col="status" :active="isColFilterActive('status')" :open="openFilterCol === 'status'"
                    @open="openColFilter('status')" @close="openFilterCol = null"
                    :draft="filterDraft" :unique-values="colUniqueValues" :filter-search="filterSearch"
                    @update:filter-search="filterSearch = $event" @toggle-value="toggleDraftValue"
                    @apply="applyDraft" @clear="clearDraft"
                  />
                </div>
              </th>

              <!-- Actions -->
              <th class="px-3 py-2.5 text-center text-xs font-semibold text-white uppercase tracking-wider">
                Actions
              </th>
            </tr>
          </thead>

          <tbody class="bg-white divide-y divide-gray-200">
            <!-- Loading state -->
            <tr v-if="loading">
              <td colspan="9" class="px-4 py-8 text-center text-sm text-gray-400">
                <div class="flex items-center justify-center gap-2">
                  <UIcon name="i-heroicons-arrow-path" class="w-4 h-4 text-orange-500 animate-spin" />
                  Loading bills…
                </div>
              </td>
            </tr>

            <!-- Empty state -->
            <tr v-else-if="paginatedBills.length === 0">
              <td colspan="9" class="px-4 py-10 text-center">
                <div class="flex flex-col items-center gap-2 text-sm text-gray-400">
                  <UIcon name="i-heroicons-document-magnifying-glass" class="w-10 h-10 text-gray-300" />
                  No bills found matching your criteria
                </div>
              </td>
            </tr>

            <!-- Bill rows -->
            <template v-else>
              <tr
                v-for="bill in paginatedBills"
                :key="bill._id"
                :class="isCancelled(bill)
                  ? 'bg-red-50/60 border-l-2 border-l-red-300'
                  : 'hover:bg-amber-50 transition-colors'"
              >
                <!-- Bill No -->
                <td class="px-3 py-2 whitespace-nowrap">
                  <span class="text-xs font-bold text-gray-900 font-mono">{{ bill.bno || '—' }}</span>
                </td>

                <!-- Date -->
                <td class="px-3 py-2 whitespace-nowrap text-xs text-gray-600">
                  {{ formatDate(bill.bdate) }}
                </td>

                <!-- Party -->
                <td class="px-3 py-2 whitespace-nowrap hidden sm:table-cell">
                  <span v-if="isCancelled(bill)" class="font-mono tracking-widest text-gray-300 text-xs select-none">████</span>
                  <span v-else class="text-xs text-gray-900">{{ getBillPartyName(bill) }}</span>
                </td>

                <!-- Type -->
                <td class="px-3 py-2 whitespace-nowrap">
                  <UBadge
                    :label="{ PURCHASE: 'PUR', CREDIT_NOTE: 'CN', DEBIT_NOTE: 'DN' }[(bill.btype || '').toUpperCase()] || 'SLS'"
                    :color="{ PURCHASE: 'purple', CREDIT_NOTE: 'warning', DEBIT_NOTE: 'neutral' }[(bill.btype || '').toUpperCase()] || 'success'"
                    variant="subtle"
                    size="sm"
                  />
                </td>

                <!-- Taxable -->
                <td class="px-3 py-2 whitespace-nowrap text-right hidden md:table-cell">
                  <span v-if="isCancelled(bill)" class="font-mono tracking-widest text-gray-300 text-xs select-none">████</span>
                  <span v-else class="text-xs font-mono text-gray-700">{{ fmtAmt(bill.gtot || 0) }}</span>
                </td>

                <!-- Tax -->
                <td class="px-3 py-2 whitespace-nowrap text-right hidden md:table-cell">
                  <span v-if="isCancelled(bill)" class="font-mono tracking-widest text-gray-300 text-xs select-none">████</span>
                  <span v-else class="text-xs font-mono text-gray-700">{{ fmtAmt(getTax(bill)) }}</span>
                </td>

                <!-- Total -->
                <td class="px-3 py-2 whitespace-nowrap text-right">
                  <span v-if="isCancelled(bill)" class="font-mono tracking-widest text-gray-300 text-xs select-none">████</span>
                  <span v-else class="text-xs font-mono font-semibold text-gray-900">{{ fmtAmt(bill.ntot || 0) }}</span>
                </td>

                <!-- Status -->
                <td class="px-3 py-2 whitespace-nowrap">
                  <UBadge
                    :label="isCancelled(bill) ? 'Cancelled' : 'Active'"
                    :color="isCancelled(bill) ? 'error' : 'success'"
                    variant="subtle"
                    size="sm"
                  />
                </td>

                <!-- Actions -->
                <td class="px-3 py-2 whitespace-nowrap text-center">
                  <div class="flex items-center justify-center gap-1">
                    <UButton
                      icon="i-heroicons-eye"
                      color="info"
                      variant="ghost"
                      size="xs"
                      title="View details"
                      @click="showBillDetails(bill._id)"
                    />
                    <UButton
                      v-if="!isCancelled(bill) && ['SALES','PURCHASE'].includes((bill.btype||'SALES').toUpperCase())"
                      icon="i-heroicons-arrow-uturn-left"
                      color="warning"
                      variant="ghost"
                      size="xs"
                      title="Create return"
                      @click="initiateReturn(bill)"
                    />
                  </div>
                </td>
              </tr>
            </template>
          </tbody>
        </table>
      </div>

      <!-- ── Pagination ────────────────────────────────────────────────────────── -->
      <div class="flex flex-col sm:flex-row items-center justify-between gap-3 px-4 py-3 border-t border-gray-200 bg-gray-50">
        <div class="flex items-center gap-2">
          <p class="text-xs text-gray-500">
            Showing
            {{ sortedBills.length ? (currentPage - 1) * itemsPerPage + 1 : 0 }}–{{
              Math.min(currentPage * itemsPerPage, sortedBills.length)
            }}
            of {{ sortedBills.length }} bills
          </p>
          <USelect
            v-model="itemsPerPage"
            :items="itemsPerPageOptions"
            size="xs"
            class="w-28"
            @update:model-value="currentPage = 1"
          />
        </div>

        <UPagination
          v-model:page="currentPage"
          :total="sortedBills.length"
          :page-count="itemsPerPage"
          size="xs"
          :max="7"
          :ui="{ list: 'gap-1' }"
        />
      </div>
    </div>


    <!-- ════════════════════════════════════════════════════════════════════════
         Bill Detail Modal
    ═══════════════════════════════════════════════════════════════════════════ -->
    <UModal
      v-model:open="modalOpen"
      :title="detailBill ? `Bill #${detailBill.bno}` : 'Bill Details'"
      description="Detailed breakdown of the selected bill"
      :ui="{ content: 'max-w-3xl' }"
    >
      <template #body>
        <!-- Loading -->
        <div v-if="detailLoading" class="flex items-center justify-center py-12 gap-3 text-sm text-gray-400">
          <UIcon name="i-heroicons-arrow-path" class="w-5 h-5 animate-spin text-orange-500" />
          Loading bill details…
        </div>

        <div v-else-if="detailBill" class="space-y-5">
          <!-- Meta row -->
          <div class="flex flex-wrap gap-3 items-center">
            <span class="font-mono text-xl font-bold text-gray-900">{{ detailBill.bno }}</span>
            <UBadge
              :label="{ PURCHASE: 'PURCHASE', CREDIT_NOTE: 'CREDIT NOTE', DEBIT_NOTE: 'DEBIT NOTE' }[(detailBill.btype||'').toUpperCase()] || 'SALES'"
              :color="{ PURCHASE: 'purple', CREDIT_NOTE: 'warning', DEBIT_NOTE: 'neutral' }[(detailBill.btype||'').toUpperCase()] || 'success'"
              variant="subtle"
            />
            <UBadge
              :label="isCancelled(detailBill) ? 'CANCELLED' : 'ACTIVE'"
              :color="isCancelled(detailBill) ? 'error' : 'success'"
              variant="solid"
            />
            <span class="text-sm text-gray-500 ml-auto">{{ formatDate(detailBill.bdate) }}</span>
          </div>

          <!-- Reference bill link -->
          <div v-if="detailBill.ref_bill_id" class="text-xs">
            <button class="text-blue-600 hover:underline font-bold" @click="showBillDetails(detailBill.ref_bill_id)">
              ↩ Original Bill: {{ detailBill.ref_bill_id }}
            </button>
          </div>

          <USeparator />

          <!-- Party info -->
          <div class="grid grid-cols-2 gap-x-6 gap-y-2">
            <div>
              <p class="text-[10px] font-bold uppercase tracking-wide text-gray-400">Party</p>
              <p class="text-sm font-semibold text-gray-900">{{ getBillPartyName(detailBill) }}</p>
            </div>
            <div>
              <p class="text-[10px] font-bold uppercase tracking-wide text-gray-400">GSTIN</p>
              <p class="text-sm font-mono text-gray-900">{{ detailBill.gstin || 'UNREGISTERED' }}</p>
            </div>
            <div>
              <p class="text-[10px] font-bold uppercase tracking-wide text-gray-400">Address</p>
              <p class="text-sm text-gray-700">{{ detailBill.addr || '—' }}</p>
            </div>
            <div>
              <p class="text-[10px] font-bold uppercase tracking-wide text-gray-400">State</p>
              <p class="text-sm text-gray-700">{{ detailBill.state || '—' }}</p>
            </div>
          </div>

          <USeparator />

          <!-- Items table -->
          <div class="rounded-lg border border-gray-200 overflow-hidden">
            <table class="min-w-full divide-y divide-gray-200 text-sm">
              <thead class="bg-gray-50">
                <tr>
                  <th class="px-3 py-2 text-left text-xs font-semibold text-gray-500 uppercase">Item</th>
                  <th class="px-3 py-2 text-left text-xs font-semibold text-gray-500 uppercase">Batch</th>
                  <th class="px-3 py-2 text-right text-xs font-semibold text-gray-500 uppercase">Qty</th>
                  <th class="px-3 py-2 text-right text-xs font-semibold text-gray-500 uppercase">Rate</th>
                  <th class="px-3 py-2 text-right text-xs font-semibold text-gray-500 uppercase">Total</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-gray-100">
                <template v-if="detailBill.items?.length">
                  <tr v-for="(item, i) in detailBill.items" :key="i" class="hover:bg-gray-50">
                    <td class="px-3 py-2 text-xs font-medium text-gray-900">{{ item.item || '—' }}</td>
                    <td class="px-3 py-2 text-xs text-gray-500">
                      {{ (item.item_type || 'GOODS') === 'SERVICE' ? '—' : (item.batch || '—') }}
                    </td>
                    <td class="px-3 py-2 text-xs text-right font-mono text-gray-700">
                      {{ (item.item_type || 'GOODS') !== 'SERVICE' || item.show_qty !== false ? (item.qty || 0) : '—' }}
                    </td>
                    <td class="px-3 py-2 text-xs text-right font-mono text-gray-700">{{ fmtAmt(item.rate || 0) }}</td>
                    <td class="px-3 py-2 text-xs text-right font-mono font-semibold text-gray-900">{{ fmtAmt(item.total || 0) }}</td>
                  </tr>
                </template>
                <tr v-else>
                  <td colspan="5" class="px-3 py-4 text-center text-xs text-gray-400">No items found</td>
                </tr>
              </tbody>
            </table>
          </div>

          <!-- Other charges -->
          <div v-if="detailBill.otherCharges?.length" class="space-y-1">
            <p class="text-[10px] font-bold uppercase tracking-wide text-gray-400">Other Charges</p>
            <div
              v-for="(charge, i) in detailBill.otherCharges"
              :key="i"
              class="flex justify-between items-center px-3 py-2 bg-gray-50 rounded border border-gray-100 text-xs"
            >
              <span class="font-medium text-gray-700">{{ charge.name || charge.type }}</span>
              <span class="font-bold text-gray-900 font-mono">{{ fmtAmt(charge.amount || 0) }}</span>
            </div>
          </div>

          <!-- Totals -->
          <div class="rounded-lg bg-gray-50 border border-gray-200 p-4 space-y-2">
            <div class="flex justify-between text-xs text-gray-600">
              <span>Taxable Amount</span>
              <span class="font-mono">{{ fmtAmt(detailBill.gtot || 0) }}</span>
            </div>
            <div v-if="detailBill.cgst" class="flex justify-between text-xs text-gray-600">
              <span>CGST</span>
              <span class="font-mono">{{ fmtAmt(detailBill.cgst) }}</span>
            </div>
            <div v-if="detailBill.sgst" class="flex justify-between text-xs text-gray-600">
              <span>SGST</span>
              <span class="font-mono">{{ fmtAmt(detailBill.sgst) }}</span>
            </div>
            <div v-if="detailBill.igst" class="flex justify-between text-xs text-gray-600">
              <span>IGST</span>
              <span class="font-mono">{{ fmtAmt(detailBill.igst) }}</span>
            </div>
            <div class="flex justify-between text-xs text-gray-600">
              <span>Total Tax</span>
              <span class="font-mono">{{ fmtAmt(getTax(detailBill)) }}</span>
            </div>
            <USeparator />
            <div class="flex justify-between text-base font-bold text-gray-900">
              <span>Grand Total</span>
              <span class="font-mono">{{ fmtAmt(detailBill.ntot || 0) }}</span>
            </div>
          </div>

          <!-- Cancel section -->
          <div v-if="!isCancelled(detailBill)" class="rounded-lg border border-red-200 overflow-hidden">
            <button
              class="w-full flex items-center justify-between px-4 py-2.5 bg-red-50 text-red-700 text-xs font-semibold hover:bg-red-100 transition"
              @click="cancelFormVisible = !cancelFormVisible"
            >
              <span class="flex items-center gap-2">
                <UIcon name="i-heroicons-x-circle" class="w-4 h-4" />
                Cancel this Bill
              </span>
              <UIcon :name="cancelFormVisible ? 'i-heroicons-chevron-up' : 'i-heroicons-chevron-down'" class="w-4 h-4" />
            </button>

            <div v-if="cancelFormVisible" class="p-4 space-y-3 bg-white">
              <USelect
                v-model="cancelReason"
                :items="cancelReasonOptions"
                placeholder="Select cancellation reason"
                size="sm"
              />
              <textarea
                v-model="cancelRemarks"
                rows="2"
                placeholder="Additional remarks (optional)"
                class="w-full px-3 py-2 text-sm border border-gray-300 rounded-lg focus:ring-2 focus:ring-red-400 focus:border-red-400 resize-none"
              />
              <div class="flex gap-2">
                <UButton
                  icon="i-heroicons-check"
                  label="Confirm Cancellation"
                  color="error"
                  size="sm"
                  :loading="cancelling"
                  :disabled="!cancelReason"
                  @click="cancelBill"
                />
                <UButton
                  label="Discard"
                  color="neutral"
                  variant="outline"
                  size="sm"
                  @click="cancelFormVisible = false; cancelReason = ''; cancelRemarks = ''"
                />
              </div>
            </div>
          </div>
        </div>
      </template>

      <template #footer>
        <div class="flex w-full items-center justify-between gap-2">
          <div class="flex gap-2">
            <UButton
              v-if="detailBill"
              icon="i-heroicons-document-arrow-down"
              label="PDF"
              color="primary"
              variant="subtle"
              size="sm"
              @click="downloadBillPdf(detailBill)"
            />
            <UButton
              v-if="detailBill"
              icon="i-heroicons-table-cells"
              label="Excel"
              color="success"
              variant="subtle"
              size="sm"
              @click="downloadBillExcel(detailBill)"
            />
            <UButton
              v-if="detailBill"
              icon="i-heroicons-printer"
              label="Print"
              color="neutral"
              variant="outline"
              size="sm"
              @click="() => typeof window !== 'undefined' && window.print()"
            />
            <UButton
              v-if="detailBill && !isCancelled(detailBill)"
              icon="i-heroicons-pencil-square"
              label="Edit"
              color="warning"
              variant="subtle"
              size="sm"
              @click="editBill(detailBill)"
            />
          </div>
          <UButton
            label="Close"
            color="neutral"
            variant="outline"
            size="sm"
            @click="modalOpen = false"
          />
        </div>
      </template>
    </UModal>

  </div>
</template>