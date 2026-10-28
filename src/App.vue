<script setup>
import { ref, computed, onMounted, onBeforeUnmount, nextTick, watch } from 'vue'

const DEMO_STORAGE_KEY = 'hotel-sales-management-demo-sales'
const DEMO_STAFF_STORAGE_KEY = 'hotel-sales-management-demo-staff'
const DEMO_ACCOUNT_TYPES = [
  { id: 1, type_name: 'New' },
  { id: 2, type_name: 'Existing' },
  { id: 3, type_name: 'Growth' }
]
const DEMO_SALES = [
  {
    id: 1, salesperson_id: 1, salesperson: 'Alex Tan', company_name: 'Demo Events Sdn. Bhd.',
    company_type_id: 1, company_type: 'Corporate', source_id: 2, source: 'Banquet',
    event_type: 'Annual Dinner', account_type_id: 2, account_type: 'Existing',
    revenue: 12800, date_in: '2026-01-14', date_out: '2026-01-15', pax: 180, rooms_count: 12, rate_quoted: 14500
  },
  {
    id: 2, salesperson_id: 2, salesperson: 'Maya Lee', company_name: 'Sample Tech Group',
    company_type_id: 1, company_type: 'Corporate', source_id: 1, source: 'Room',
    event_type: 'Training', account_type_id: 1, account_type: 'New',
    revenue: 8600, date_in: '2026-02-08', date_out: '2026-02-10', pax: 42, rooms_count: 21, rate_quoted: 9200
  },
  {
    id: 3, salesperson_id: 1, salesperson: 'Alex Tan', company_name: 'City Community Office',
    company_type_id: 2, company_type: 'Government', source_id: 3, source: 'Other',
    event_type: 'Public Seminar', account_type_id: 3, account_type: 'Growth',
    revenue: 5400, date_in: '2026-03-21', date_out: '2026-03-21', pax: 95, rooms_count: 0, rate_quoted: 6000
  }
]
const DEMO_STAFF = [
  { id: 1, name: 'Alex Tan' },
  { id: 2, name: 'Maya Lee' }
]
const DESKTOP_SALES_PAGE_SIZE = 4
const MOBILE_SALES_PAGE_SIZE = 5

const rawSalesData = ref([])
const salespersonData = ref([])
const accountTypeData = ref([])
const currentSalesPage = ref(1)
const isMobileViewport = ref(false)
const salesView = ref('cards')
const isPrintingReport = ref(false)
const saleEntryCard = ref(null)
const salesListCard = ref(null)
const editPlaceholderHeight = ref(0)
const salespersonPicker = ref(null)
const isSalespersonListOpen = ref(false)
const pendingSalespersonRemoval = ref(null)
const pendingSaleRemoval = ref(null)
const removeSalespersonCancelButton = ref(null)

const filterSalesperson = ref('')
const filterSource = ref('')
const filterMonth = ref('')

const form = ref({
  id: '',
  salesperson_id: '',
  company_name: '',
  company_type_id: '1',
  source_id: '1',
  event_type: '',
  account_type_id: '2',
  revenue: '',
  date_in: '',
  date_out: '',
  pax: '',
  rooms_count: '',
  rate_quoted: ''
})

const isEditing = ref(false)
const newSalespersonName = ref('')
const salespersonAddStatus = ref('')

const dashboardYear = ref('')
const dashboardSalesperson = ref('')
const dashboardMonth = ref('')
const hoveredBarMonth = ref('')
const hoveredCategory = ref('')
const authLoading = ref(true)
const currentUser = ref(null)
const authBusy = ref(false)
const authError = ref('')
const authForm = ref({ username: '', password: '' })
const showPassword = ref(false)

// Fetch Data on mount
let saleEntryResizeObserver

function syncSalesListPanelHeight() {
  isMobileViewport.value = window.matchMedia('(max-width: 700px)').matches
  if (isEditing.value) return
  if (!saleEntryCard.value || !salesListCard.value) return
  if (window.matchMedia('(max-width: 900px)').matches) {
    salesListCard.value.style.height = ''
    return
  }
  salesListCard.value.style.height = `${saleEntryCard.value.getBoundingClientRect().height}px`
}

watch(salesView, syncSalesListPanelHeight)
watch(isEditing, async () => {
  await nextTick()
  syncSalesListPanelHeight()
})
onMounted(async () => {
  window.addEventListener('resize', syncSalesListPanelHeight)
  document.addEventListener('click', closeSalespersonListOnOutsideClick)
  await refreshAuth()
  if (currentUser.value) {
    await loadSales()
    await nextTick()
    observeWorkspaceLayout()
  }
})

onBeforeUnmount(() => {
  saleEntryResizeObserver?.disconnect()
  window.removeEventListener('resize', syncSalesListPanelHeight)
  document.removeEventListener('click', closeSalespersonListOnOutsideClick)
})

function closeSalespersonListOnOutsideClick(event) {
  if (!salespersonPicker.value?.contains(event.target)) isSalespersonListOpen.value = false
}

function getDefaultAccountTypeId() {
  const existingType = accountTypeData.value.find(
    accountType => String(accountType.type_name).toLowerCase() === 'existing'
  )
  return String(existingType?.id || accountTypeData.value[0]?.id || '')
}

async function loadSales() {
  const savedSales = localStorage.getItem(DEMO_STORAGE_KEY)
  const savedStaff = localStorage.getItem(DEMO_STAFF_STORAGE_KEY)
  rawSalesData.value = savedSales ? JSON.parse(savedSales) : structuredClone(DEMO_SALES)
  salespersonData.value = savedStaff ? JSON.parse(savedStaff) : structuredClone(DEMO_STAFF)
  accountTypeData.value = DEMO_ACCOUNT_TYPES
  if (salespersonData.value.length > 0 && !form.value.salesperson_id) {
    form.value.salesperson_id = String(salespersonData.value[0].id)
  }
  if (!accountTypeData.value.some(accountType => String(accountType.id) === form.value.account_type_id)) {
    form.value.account_type_id = getDefaultAccountTypeId()
  }
}

function observeWorkspaceLayout() {
  if (!saleEntryResizeObserver) saleEntryResizeObserver = new ResizeObserver(syncSalesListPanelHeight)
  if (saleEntryCard.value) saleEntryResizeObserver.observe(saleEntryCard.value)
  syncSalesListPanelHeight()
}

async function refreshAuth() {
  currentUser.value = sessionStorage.getItem('hotel-demo-user')
    ? { username: sessionStorage.getItem('hotel-demo-user') }
    : null
  authLoading.value = false
}

async function submitAuth() {
  authBusy.value = true
  authError.value = ''
  if (authForm.value.username === 'demo' && authForm.value.password === 'demo') {
    currentUser.value = { username: 'demo' }
    sessionStorage.setItem('hotel-demo-user', 'demo')
    authForm.value.password = ''
    await loadSales()
    await nextTick()
    observeWorkspaceLayout()
  } else {
    authError.value = 'Use the demo credentials shown below.'
  }
  authBusy.value = false
}

async function logout() {
  sessionStorage.removeItem('hotel-demo-user')
  currentUser.value = null
  rawSalesData.value = []
  salespersonData.value = []
}

// Filtered sales list
const salespersonFilterOptions = computed(() => {
  return salespersonData.value
    .map(person => person.name)
    .sort((a, b) => a.localeCompare(b))
})

const filteredSales = computed(() => {
  return rawSalesData.value.filter(s => {
    const matchPerson = !filterSalesperson.value || s.salesperson === filterSalesperson.value
    const matchSource = !filterSource.value || String(s.source_id) === filterSource.value
    const saleMonth = s.date_in ? s.date_in.substring(0, 7) : ''
    const matchMonth = !filterMonth.value || saleMonth === filterMonth.value
    return matchPerson && matchSource && matchMonth
  })
})

const salesPageSize = computed(() => (
  isMobileViewport.value ? MOBILE_SALES_PAGE_SIZE : DESKTOP_SALES_PAGE_SIZE
))

const totalPages = computed(() => Math.max(1, Math.ceil(filteredSales.value.length / salesPageSize.value)))

const paginatedSales = computed(() => {
  if (isPrintingReport.value) return filteredSales.value
  const start = (currentSalesPage.value - 1) * salesPageSize.value
  return filteredSales.value.slice(start, start + salesPageSize.value)
})

const paginationStatusText = computed(() => {
  if (filteredSales.value.length === 0) {
    return `Page ${currentSalesPage.value} of ${totalPages.value} · Showing 0 of 0 records`
  }
  const start = (currentSalesPage.value - 1) * salesPageSize.value + 1
  const end = Math.min(currentSalesPage.value * salesPageSize.value, filteredSales.value.length)
  return `Page ${currentSalesPage.value} of ${totalPages.value} · Showing ${start}–${end} of ${filteredSales.value.length} records`
})

function changeSalesPage(delta) {
  currentSalesPage.value = Math.min(Math.max(1, currentSalesPage.value + delta), totalPages.value)
}

watch(totalPages, pageCount => {
  if (currentSalesPage.value > pageCount) currentSalesPage.value = pageCount
})

function resetFilters() {
  filterSalesperson.value = ''
  filterSource.value = ''
  filterMonth.value = ''
  currentSalesPage.value = 1
}

function setSalesView(view) {
  salesView.value = view
}

// Form Date In change auto-fills Date Out
function handleDateInChange(e) {
  if (!form.value.date_out) {
    form.value.date_out = e.target.value
  }
}

function formatDateDisplay(dateStr) {
  if (!dateStr) return '-'
  const date = dateStr.split('T')[0]
  const match = date.match(/^(\d{4})-(\d{2})-(\d{2})$/)
  if (!match) return date
  const months = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec']
  return `${Number(match[3])}-${months[Number(match[2]) - 1]}`
}

function formatSaleDateRange(dateIn, dateOut) {
  const format = dateStr => {
    if (!dateStr) return ''
    const match = dateStr.split('T')[0].match(/^(\d{4})-(\d{2})-(\d{2})$/)
    return match ? `${match[3]}/${match[2]}/${match[1].slice(-2)}` : dateStr
  }

  const start = format(dateIn)
  const end = format(dateOut)
  if (!start) return end || '-'
  if (!end || dateIn.split('T')[0] === dateOut.split('T')[0]) return start
  return `${start} - ${end}`
}

async function handleFormSubmit() {
  const payload = { ...form.value }
  const salesperson = salespersonData.value.find(person => String(person.id) === String(payload.salesperson_id))
  const accountType = accountTypeData.value.find(type => String(type.id) === String(payload.account_type_id))
  const companyTypes = { '1': 'Corporate', '2': 'Government', '3': 'Others' }
  const sources = { '1': 'Room', '2': 'Banquet', '3': 'Other' }
  const record = {
    ...payload,
    id: isEditing.value ? payload.id : Date.now(),
    salesperson: salesperson?.name || 'Demo Salesperson',
    account_type: accountType?.type_name || 'Existing',
    company_type: companyTypes[payload.company_type_id] || 'Others',
    source: sources[payload.source_id] || 'Other'
  }
  rawSalesData.value = isEditing.value
    ? rawSalesData.value.map(sale => sale.id === payload.id ? record : sale)
    : [record, ...rawSalesData.value]
  localStorage.setItem(DEMO_STORAGE_KEY, JSON.stringify(rawSalesData.value))
  alert(isEditing.value ? 'Demo record updated!' : 'Demo record created!')
  resetForm()
}

function populateEditForm(id) {
  const item = rawSalesData.value.find(s => s.id === id)
  if (!item) return
  const formRect = saleEntryCard.value?.getBoundingClientRect()
  editPlaceholderHeight.value = formRect?.height || 0
  form.value = {
    id: item.id,
    salesperson_id: String(item.salesperson_id || ''),
    company_name: item.company_name || '',
    company_type_id: String(item.company_type_id || '1'),
    source_id: String(item.source_id || '1'),
    event_type: item.event_type || '',
    account_type_id: String(item.account_type_id || '2'),
    revenue: item.revenue || '',
    date_in: item.date_in ? item.date_in.split('T')[0] : '',
    date_out: item.date_out ? item.date_out.split('T')[0] : '',
    pax: item.pax || '',
    rooms_count: item.rooms_count || '',
    rate_quoted: item.rate_quoted || ''
  }
  isEditing.value = true
}

function resetForm() {
  form.value = {
    id: '',
    salesperson_id: salespersonData.value[0]?.id ? String(salespersonData.value[0].id) : '',
    company_name: '',
    company_type_id: '1',
    source_id: '1',
    event_type: '',
    account_type_id: getDefaultAccountTypeId(),
    revenue: '',
    date_in: '',
    date_out: '',
    pax: '',
    rooms_count: '',
    rate_quoted: ''
  }
  isEditing.value = false
}

async function requestSaleRemoval(id) {
  const sale = rawSalesData.value.find(item => String(item.id) === String(id))
  if (!sale) return
  pendingSaleRemoval.value = sale
  await nextTick()
  removeSalespersonCancelButton.value?.focus()
}

function cancelSaleRemoval() {
  pendingSaleRemoval.value = null
}

async function deleteSale() {
  const sale = pendingSaleRemoval.value
  if (!sale) return
  pendingSaleRemoval.value = null
  rawSalesData.value = rawSalesData.value.filter(item => item.id !== sale.id)
  localStorage.setItem(DEMO_STORAGE_KEY, JSON.stringify(rawSalesData.value))
}

async function addSalesperson() {
  const name = newSalespersonName.value.trim()
  if (!name) return
  const data = { id: Date.now(), name }
  salespersonData.value.push(data)
  localStorage.setItem(DEMO_STAFF_STORAGE_KEY, JSON.stringify(salespersonData.value))
  form.value.salesperson_id = String(data.id)
  newSalespersonName.value = ''
  salespersonAddStatus.value = `${data.name} added to this browser demo!`
}

async function requestSalespersonRemoval(salespersonId) {
  const id = String(salespersonId || '')
  const salesperson = salespersonData.value.find(person => String(person.id) === id)
  if (!salesperson) return
  pendingSalespersonRemoval.value = salesperson
  isSalespersonListOpen.value = false
  await nextTick()
  removeSalespersonCancelButton.value?.focus()
}

function cancelSalespersonRemoval() {
  pendingSalespersonRemoval.value = null
  nextTick(() => salespersonPicker.value?.querySelector('.salesperson-picker-trigger')?.focus())
}

async function removeSalesperson() {
  const salesperson = pendingSalespersonRemoval.value
  if (!salesperson) return
  const id = String(salesperson.id)
  pendingSalespersonRemoval.value = null

  salespersonAddStatus.value = ''
  salespersonData.value = salespersonData.value.filter(person => String(person.id) !== id)
  localStorage.setItem(DEMO_STAFF_STORAGE_KEY, JSON.stringify(salespersonData.value))
  if (filterSalesperson.value === salesperson.name) filterSalesperson.value = ''
  if (dashboardSalesperson.value === salesperson.name) dashboardSalesperson.value = ''
  if (!isEditing.value && String(form.value.salesperson_id) === id) {
    form.value.salesperson_id = salespersonData.value[0] ? String(salespersonData.value[0].id) : ''
  }
  salespersonAddStatus.value = `${salesperson.name} archived in this browser demo.`
  nextTick(() => salespersonPicker.value?.querySelector('.salesperson-picker-trigger')?.focus())
}

function printSalesReport() {
  isPrintingReport.value = true
  salesView.value = 'table'
  nextTick(() => {
    window.print()
    setTimeout(() => {
      isPrintingReport.value = false
      salesView.value = 'cards'
    }, 500)
  })
}

// Summary computations for table footer
const tableTotals = computed(() => {
  let pax = 0, rooms = 0, roomRev = 0, banquetRev = 0, otherRev = 0, confirm = 0
  filteredSales.value.forEach(s => {
    pax += Number(s.pax) || 0
    rooms += Number(s.rooms_count) || 0
    const rev = Number(s.revenue) || 0
    confirm += rev
    const src = String(s.source_id)
    if (src === '1') roomRev += rev
    else if (src === '2') banquetRev += rev
    else otherRev += rev
  })
  return { pax, rooms, roomRev, banquetRev, otherRev, confirm }
})

const reportSalespersonName = computed(() => filterSalesperson.value)

const reportPeriodLabel = computed(() => {
  if (!filterMonth.value) return 'All Dates'
  const [year, month] = filterMonth.value.split('-')
  const lastDay = new Date(Number(year), Number(month), 0).getDate()
  return `${1}-${lastDay}/${Number(month)}/${year}`
})

const reportAccountTypeCounts = computed(() => {
  const counts = { existing: 0, new: 0, growth: 0 }
  filteredSales.value.forEach(sale => {
    const type = (sale.account_type || '').toLowerCase()
    if (type === 'new') counts.new++
    else if (type === 'growth') counts.growth++
    else counts.existing++
  })
  return counts
})

// Dashboard computations
const availableYears = computed(() => {
  const years = [...new Set(rawSalesData.value.filter(s => s.date_in).map(s => s.date_in.substring(0, 4)))]
  return years.sort()
})

const currentDashboardYear = computed({
  get() {
    if (availableYears.value.includes(dashboardYear.value)) return dashboardYear.value
    return availableYears.value[availableYears.value.length - 1] || ''
  },
  set(val) { dashboardYear.value = val }
})

const dashboardFilteredSales = computed(() => {
  return rawSalesData.value.filter(s => !dashboardSalesperson.value || s.salesperson === dashboardSalesperson.value)
})

const monthlyBarData = computed(() => {
  const totals = Array(12).fill(0)
  const transactions = Array(12).fill(0)
  const yr = currentDashboardYear.value
  rawSalesData.value.forEach(s => {
    if (s.date_in && s.date_in.substring(0, 4) === yr) {
      const m = Number(s.date_in.substring(5, 7)) - 1
      if (m >= 0 && m < 12) {
        totals[m] += Number(s.revenue) || 0
        transactions[m]++
      }
    }
  })
  const max = Math.max(...totals, 1)
  return totals.map((val, idx) => ({
    month: ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov','Dec'][idx],
    revenue: val,
    transactions: transactions[idx],
    height: val > 0 ? Math.max((val / max) * 100, 8) : 0
  }))
})

const monthlyLineChart = computed(() => {
  const chartWidth = 720
  const chartHeight = 220
  const chartTop = 12
  const chartBottom = 32
  const plotHeight = chartHeight - chartTop - chartBottom
  const points = monthlyBarData.value
  const maxRevenue = Math.max(...points.map(point => point.revenue), 1)
  const xStep = chartWidth / (points.length - 1)

  return {
    points: points.map((point, index) => ({
      ...point,
      x: index * xStep,
      y: chartTop + plotHeight - (point.revenue / maxRevenue) * plotHeight
    })),
    line: points.map((point, index) => {
      const x = index * xStep
      const y = chartTop + plotHeight - (point.revenue / maxRevenue) * plotHeight
      return `${x},${y}`
    }).join(' '),
    baseline: chartTop + plotHeight
  }
})

const availableMonths = [
  { value: '01', name: 'January' }, { value: '02', name: 'February' },
  { value: '03', name: 'March' }, { value: '04', name: 'April' },
  { value: '05', name: 'May' }, { value: '06', name: 'June' },
  { value: '07', name: 'July' }, { value: '08', name: 'August' },
  { value: '09', name: 'September' }, { value: '10', name: 'October' },
  { value: '11', name: 'November' }, { value: '12', name: 'December' }
]

const currentDashboardMonth = computed({
  get() {
    const salesForYear = dashboardFilteredSales.value.filter(
      s => s.date_in && s.date_in.substring(0, 4) === currentDashboardYear.value
    )
    if (dashboardMonth.value && salesForYear.some(s => s.date_in.substring(5, 7) === dashboardMonth.value)) {
      return dashboardMonth.value
    }
    return salesForYear.map(s => s.date_in.substring(5, 7)).sort().pop() || '01'
  },
  set(val) { dashboardMonth.value = val }
})

const selectedMonthSales = computed(() => {
  const yr = currentDashboardYear.value
  const mo = currentDashboardMonth.value
  return dashboardFilteredSales.value.filter(s => s.date_in && s.date_in.substring(0, 7) === `${yr}-${mo}`)
})

const monthlyRevenue = computed(() => selectedMonthSales.value.reduce((sum, s) => sum + (Number(s.revenue) || 0), 0))

const topClient = computed(() => {
  const map = {}
  selectedMonthSales.value.forEach(s => {
    map[s.company_name] = (map[s.company_name] || 0) + (Number(s.revenue) || 0)
  })
  const sorted = Object.entries(map).sort((a, b) => b[1] - a[1])
  return sorted[0] ? { name: sorted[0][0], rev: sorted[0][1] } : { name: 'None', rev: 0 }
})

const accountTypeCounts = computed(() => {
  const counts = { existing: 0, new: 0, growth: 0 }
  selectedMonthSales.value.forEach(s => {
    const type = (s.account_type || '').toLowerCase()
    if (type === 'new') counts.new++
    else if (type === 'growth') counts.growth++
    else counts.existing++
  })
  return counts
})

const dashboardPeriodLabel = computed(() => {
  const monthName = availableMonths.find(month => month.value === currentDashboardMonth.value)?.name
  return monthName ? `${monthName.slice(0, 3)} ${currentDashboardYear.value}` : '-'
})

const categoryRevenue = computed(() => {
  const totals = new Map()
  selectedMonthSales.value.forEach(sale => {
    const category = sale.company_type || 'Others'
    totals.set(category, (totals.get(category) || 0) + (Number(sale.revenue) || 0))
  })

  const entries = [...totals.entries()].filter(([, revenue]) => revenue > 0)
  const totalRevenue = entries.reduce((sum, [, revenue]) => sum + revenue, 0)
  const colors = {
    Corporate: '#00B8C4',
    Gov: '#C6FF00',
    Government: '#C6FF00',
    Others: '#F5F5F5'
  }

  return entries.map(([category, revenue]) => ({
    category,
    revenue,
    percentage: (revenue / totalRevenue) * 100,
    color: colors[category] || '#D5D5DC'
  }))
})

const categoryChartSlices = computed(() => {
  let cumulativePercent = 0
  return categoryRevenue.value.map(entry => {
    const startAngle = (cumulativePercent / 100) * 360 - 90
    cumulativePercent += entry.percentage
    const endAngle = (cumulativePercent / 100) * 360 - 90
    if (entry.percentage >= 99.999) {
      return { ...entry, fullCircle: true, path: '' }
    }
    const startX = 85 + 85 * Math.cos((startAngle * Math.PI) / 180)
    const startY = 85 + 85 * Math.sin((startAngle * Math.PI) / 180)
    const endX = 85 + 85 * Math.cos((endAngle * Math.PI) / 180)
    const endY = 85 + 85 * Math.sin((endAngle * Math.PI) / 180)
    const largeArc = entry.percentage > 50 ? 1 : 0
    return {
      ...entry,
      fullCircle: false,
      path: `M 85 85 L ${startX} ${startY} A 85 85 0 ${largeArc} 1 ${endX} ${endY} Z`
    }
  })
})

const categoryChartLabel = computed(() => `Revenue by category: ${categoryRevenue.value
  .map(({ category, revenue }) => `${category} RM${revenue.toFixed(2)}`)
  .join(', ')}`)

const hoveredCategoryData = computed(() => categoryRevenue.value.find(
  entry => entry.category === hoveredCategory.value
))
</script>

<template>
  <main v-if="authLoading" class="auth-loading" aria-live="polite">
    <span class="auth-loading-mark" aria-hidden="true">PA</span>
    <p>Checking your workspace...</p>
  </main>

  <main v-else-if="!currentUser" class="login-page">
    <section class="login-form-wrap">
      <h2 class="login-brand-name">DEMO HOTEL</h2>
      <p class="login-subtitle">Sales Records Management</p>
      <p class="login-demo-hint">Public demo: use <strong>demo</strong> / <strong>demo</strong></p>
      <form class="login-form" @submit.prevent="submitAuth">
        <div class="login-field">
          <label for="auth-username">Username</label>
          <input id="auth-username" v-model="authForm.username" type="text" autocomplete="username" required>
        </div>
        <div class="login-field">
          <label for="auth-password">Password</label>
          <div class="password-input-wrap">
            <input id="auth-password" v-model="authForm.password" :type="showPassword ? 'text' : 'password'" autocomplete="current-password" required>
            <button
              class="password-toggle"
              type="button"
              :aria-label="showPassword ? 'Hide password' : 'Show password'"
              :aria-pressed="showPassword"
              @click="showPassword = !showPassword"
            >
              <svg v-if="showPassword" viewBox="0 0 24 24" aria-hidden="true">
                <path d="M3 3l18 18M10.6 10.6a2 2 0 0 0 2.8 2.8M9.9 5.2A10.7 10.7 0 0 1 12 5c5.5 0 9 7 9 7a17.7 17.7 0 0 1-3.1 3.8M6.2 6.2C3.9 7.7 3 12 3 12s3.5 7 9 7a10.7 10.7 0 0 0 4.1-.8" />
              </svg>
              <svg v-else viewBox="0 0 24 24" aria-hidden="true">
                <path d="M2.5 12s3.5-7 9.5-7 9.5 7 9.5 7-3.5 7-9.5 7-9.5-7-9.5-7Z" />
                <circle cx="12" cy="12" r="2.5" />
              </svg>
            </button>
          </div>
        </div>
        <p v-if="authError" class="login-error" role="alert">{{ authError }}</p>
        <button class="login-submit" type="submit" :disabled="authBusy">
          {{ authBusy ? 'Please wait...' : 'Login' }}
        </button>
      </form>
    </section>
  </main>

  <div v-else class="app-shell">
    <div class="main-content">
      <header class="page-header">
        <div class="page-header-copy">
          <h1 id="pageTitle">Demo Hotel</h1>
          <p class="page-subtitle">Sales records and performance overview.</p>
        </div>
        <button class="logout-button" type="button" @click="logout">Logout</button>
      </header>

      <div class="workspace-top">
        <!-- FORM: Record / Edit Sale -->
        <section
          v-if="isEditing"
          class="workspace-panel edit-form-placeholder"
          aria-hidden="true"
        >
          <div
            class="card"
            :style="{
              height: `${editPlaceholderHeight}px`,
              minHeight: '0',
              boxSizing: 'border-box',
              overflow: 'hidden'
            }"
          >
            <h2>Log New Sales</h2>
            <form class="sales-form-placeholder" inert>
              <div>
                <label>Salesperson</label>
                <select disabled><option>Select salesperson</option></select>
                <div class="salesperson-add">
                  <input type="text" placeholder="New salesperson name" disabled>
                  <button type="button" disabled>Add</button>
                </div>
              </div>
              <div>
                <label>Company / Client Name</label>
                <input type="text" placeholder="e.g. Public Bank Berhad" disabled>
              </div>
              <div>
                <label>Event Description</label>
                <input type="text" placeholder="e.g. Seminar / Annual Dinner" disabled>
              </div>
              <div>
                <label>Date In</label>
                <input type="date" disabled>
              </div>
              <div>
                <label>Date Out</label>
                <input type="date" disabled>
              </div>
              <div>
                <label>Pax (Guests Count)</label>
                <input type="number" placeholder="0" disabled>
              </div>
              <div>
                <label>Rooms Count</label>
                <input type="number" placeholder="0" disabled>
              </div>
              <div>
                <label>Rates Quoted (RM)</label>
                <input type="number" placeholder="0.00" disabled>
              </div>
              <div>
                <label>Revenue Source Category</label>
                <select disabled><option>Select source</option></select>
              </div>
              <div>
                <label>Revenue Amount (RM)</label>
                <input type="number" placeholder="0.00" disabled>
              </div>
              <div>
                <label>Account Type (A/C Type)</label>
                <select disabled><option>Select account type</option></select>
              </div>
              <div>
                <label>Segment / Status</label>
                <select disabled><option>Select segment</option></select>
              </div>
              <div class="full-width">
                <button type="button" class="btn-submit" disabled>Submit Sale Record</button>
              </div>
            </form>
          </div>
        </section>
        <Teleport to="body" :disabled="!isEditing">
        <section
          id="formPage"
          class="workspace-panel"
          :class="{ 'sale-edit-modal': isEditing }"
          :role="isEditing ? 'dialog' : undefined"
          :aria-modal="isEditing ? 'true' : undefined"
          :aria-labelledby="isEditing ? 'sale-form-title' : undefined"
          :tabindex="isEditing ? -1 : undefined"
          @click.self="isEditing && resetForm()"
          @keydown.esc="isEditing && resetForm()"
        >
          <div ref="saleEntryCard" class="card">
            <div class="form-card-heading">
              <h2 id="sale-form-title">{{ isEditing ? `Edit Sale Record #${form.id}` : 'Log New Sales' }}</h2>
              <button
                v-if="isEditing"
                type="button"
                class="edit-modal-close"
                aria-label="Close edit form"
                @click="resetForm"
              >×</button>
            </div>
            <form @submit.prevent="handleFormSubmit" id="salesForm">
              <div>
                <label for="salesperson_id">Salesperson</label>
                <div ref="salespersonPicker" class="salesperson-picker" @keydown.esc.stop="isSalespersonListOpen = false">
                  <button
                    id="salesperson_id"
                    type="button"
                    class="salesperson-picker-trigger"
                    :aria-expanded="isSalespersonListOpen"
                    aria-haspopup="list"
                    aria-controls="salesperson-options"
                    @click="isSalespersonListOpen = !isSalespersonListOpen"
                    @keydown.down.prevent="isSalespersonListOpen = true"
                  >
                    <span>{{ salespersonData.find(person => String(person.id) === String(form.salesperson_id))?.name || (isEditing ? rawSalesData.find(sale => String(sale.salesperson_id) === String(form.salesperson_id))?.salesperson : '') || 'Select salesperson' }}</span>
                    <svg class="salesperson-picker-arrow" aria-hidden="true" viewBox="0 0 12 8">
                      <path d="m1 1 5 5 5-5" />
                    </svg>
                  </button>
                  <div v-if="isSalespersonListOpen" id="salesperson-options" class="salesperson-picker-options" role="list">
                    <div v-for="p in salespersonData" :key="p.id" class="salesperson-picker-option" role="listitem">
                      <button
                        type="button"
                        class="salesperson-picker-choice"
                        :aria-current="String(p.id) === String(form.salesperson_id) ? 'true' : undefined"
                        @click="form.salesperson_id = String(p.id); isSalespersonListOpen = false"
                      >{{ p.name }}</button>
                      <button
                        type="button"
                        class="btn-delete-salesperson"
                        :aria-label="`Remove ${p.name}`"
                        :title="`Remove ${p.name}`"
                        @click="requestSalespersonRemoval(p.id)"
                      >×</button>
                    </div>
                    <p v-if="salespersonData.length === 0" class="salesperson-picker-empty">No salespeople added yet.</p>
                  </div>
                </div>
                <div class="salesperson-add">
                  <input v-model="newSalespersonName" type="text" maxlength="100" placeholder="New salesperson name">
                  <button @click.prevent="addSalesperson" type="button">Add</button>
                </div>
                <p class="salesperson-add-status" role="status" aria-live="polite">{{ salespersonAddStatus }}</p>
              </div>

              <div>
                <label>Company / Client Name</label>
                <input v-model="form.company_name" type="text" placeholder="e.g. Public Bank Berhad" required>
              </div>

              <div>
                <label>Event Description</label>
                <input v-model="form.event_type" type="text" placeholder="e.g. Seminar / Annual Dinner">
              </div>

              <div>
                <label>Date In</label>
                <input v-model="form.date_in" @change="handleDateInChange" type="date" required>
              </div>

              <div>
                <label>Date Out</label>
                <input v-model="form.date_out" type="date" required>
              </div>

              <div>
                <label>Pax (Guests Count)</label>
                <input v-model.number="form.pax" type="number" placeholder="0" min="0">
              </div>

              <div>
                <label>Rooms Count</label>
                <input v-model.number="form.rooms_count" type="number" placeholder="0" min="0">
              </div>

              <div>
                <label>Rates Quoted (RM)</label>
                <input v-model.number="form.rate_quoted" type="number" step="0.01" placeholder="0.00">
              </div>

              <div>
                <label>Revenue Source Category</label>
                <select v-model="form.source_id" required>
                  <option value="1">Room</option>
                  <option value="2">Banquet</option>
                  <option value="3">Other</option>
                </select>
              </div>

              <div>
                <label>Revenue Amount (RM)</label>
                <input v-model.number="form.revenue" type="number" step="0.01" placeholder="0.00" required>
              </div>

              <div>
                <label>Account Type (A/C Type)</label>
                <select v-model="form.account_type_id" required>
                  <option v-for="accountType in accountTypeData" :key="accountType.id" :value="String(accountType.id)">
                    {{ accountType.type_name }}
                  </option>
                </select>
              </div>

              <div>
                <label>Segment / Status</label>
                <select v-model="form.company_type_id" required>
                  <option value="1">Corporate</option>
                  <option value="2">Government</option>
                  <option value="3">Others</option>
                </select>
              </div>

              <div class="full-width">
                <button type="submit" class="btn-submit">{{ isEditing ? 'Update Record' : 'Submit Sale Record' }}</button>
                <button v-if="isEditing" @click="resetForm" type="button" class="btn-cancel">Cancel Edit</button>
              </div>
            </form>
          </div>
        </section>
        </Teleport>
      

        <!-- REPORT TABLE CARD -->
        <section id="reportPage" class="workspace-panel">
          <div ref="salesListCard" class="card">
            <div class="section-heading">
              <h2>Sales record</h2>
            </div>

            <!-- FILTER BAR -->
            <div class="filter-bar">
              <div class="report-filter-controls">
                <div>
                  <label>Filter Salesperson</label>
                  <select v-model="filterSalesperson" @change="currentSalesPage = 1">
                    <option value="">All Staff</option>
                    <option v-for="name in salespersonFilterOptions" :key="name" :value="name">{{ name }}</option>
                  </select>
                </div>
                <div>
                  <label>Filter Source</label>
                  <select v-model="filterSource" @change="currentSalesPage = 1">
                    <option value="">All Sources</option>
                    <option value="1">Room</option>
                    <option value="2">Banquet</option>
                    <option value="3">Other</option>
                  </select>
                </div>
                <div>
                  <label>Filter Month</label>
                  <input v-model="filterMonth" @change="currentSalesPage = 1" type="month">
                </div>
                <div>
                  <button type="button" class="btn-reset" @click="resetFilters">Reset Filter</button>
                </div>
              </div>
            </div>

            <!-- TABLE VIEW -->
            <div class="print-report-header">
              <strong>Demo Hotel</strong>
              <strong>MONTHLY REPORT</strong>
              <div class="print-report-meta">
                <span>Post Momentum - Top 20 Actual Business Generated For The Month</span>
                <span>Name: {{ reportSalespersonName }}</span>
                <span>Date: {{ reportPeriodLabel }}</span>
              </div>
            </div>
            <div id="salesTableView" class="table-responsive" :hidden="salesView !== 'table' && !isPrintingReport">
              <table>
                <thead>
                  <tr>
                    <th rowspan="2" style="width: 30px;">#</th>
                    <th rowspan="2" style="min-width: 180px;">Company</th>
                    <th rowspan="2" class="salesperson-column">Salesperson</th>
                    <th colspan="2">Date</th>
                    <th colspan="2">Details</th>
                    <th rowspan="2">Rates Quoted</th>
                    <th colspan="4">Revenue (RM)</th>
                    <th :colspan="isPrintingReport ? 1 : 2" class="classification-heading">Classification</th>
                    <th rowspan="2" style="width: 90px;">Action</th>
                  </tr>
                  <tr>
                    <th class="sub-header">In</th>
                    <th class="sub-header">Out</th>
                    <th class="sub-header">Pax</th>
                    <th class="sub-header">Room</th>
                    <th class="sub-header">Room</th>
                    <th class="sub-header">Banquet</th>
                    <th class="sub-header">Other</th>
                    <th class="sub-header">Total Confirm</th>
                    <th class="sub-header">A/C Type</th>
                    <th class="sub-header status-column">Status</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-if="filteredSales.length === 0">
                    <td colspan="16" style="text-align:center; padding: 20px;">No sales records found matching the criteria.</td>
                  </tr>
                  <tr v-for="(s, idx) in paginatedSales" :key="s.id">
                    <td>{{ isPrintingReport ? idx + 1 : (currentSalesPage - 1) * salesPageSize + idx + 1 }}</td>
                    <td class="text-left"><strong>{{ s.company_name }}</strong><br><small style="color:var(--palette-olive);" v-if="s.event_type">{{ s.event_type }}</small></td>
                    <td class="salesperson-column">{{ s.salesperson || '-' }}</td>
                    <td>{{ formatDateDisplay(s.date_in) }}</td>
                    <td>{{ formatDateDisplay(s.date_out) }}</td>
                    <td>{{ s.pax || '-' }}</td>
                    <td>{{ s.rooms_count || '-' }}</td>
                    <td class="text-right">{{ s.rate_quoted ? Number(s.rate_quoted).toFixed(2) : '-' }}</td>
                    <td class="text-right">{{ String(s.source_id) === '1' ? Number(s.revenue).toFixed(2) : '-' }}</td>
                    <td class="text-right">{{ String(s.source_id) === '2' ? Number(s.revenue).toFixed(2) : '-' }}</td>
                    <td class="text-right">{{ String(s.source_id) === '3' ? Number(s.revenue).toFixed(2) : '-' }}</td>
                    <td class="text-right" style="font-weight: bold;">{{ Number(s.revenue).toFixed(2) }}</td>
                    <td>{{ s.account_type || 'Existing' }}</td>
                    <td class="status-column">{{ s.company_type || 'Corporate' }}</td>
                    <td>
                      <button class="btn-edit" @click="populateEditForm(s.id)">Edit</button>
                      <button class="btn-delete" @click="requestSaleRemoval(s.id)">Delete</button>
                    </td>
                  </tr>
                </tbody>
                <tfoot>
                  <tr>
                    <td colspan="5" class="label-foot">TOTAL</td>
                    <td></td>
                    <td></td>
                    <td></td>
                    <td></td>
                    <td></td>
                    <td></td>
                    <td>{{ tableTotals.confirm.toFixed(2) }}</td>
                    <td v-if="!isPrintingReport" colspan="3"></td>
                    <td v-else></td>
                  </tr>
                </tfoot>
              </table>
            </div>
            <div class="print-report-summary">
              <div class="print-account-summary">
                <strong>Monthly Summary of Account Type</strong>
                <div><span>Existing Accounts</span><strong>{{ reportAccountTypeCounts.existing }}</strong></div>
                <div><span>Growth Accounts</span><strong>{{ reportAccountTypeCounts.growth }}</strong></div>
                <div><span>New Accounts</span><strong>{{ reportAccountTypeCounts.new }}</strong></div>
              </div>
              <div class="print-revenue-summary">
                <div><span>Room Revenue Generated:</span><strong>RM{{ tableTotals.roomRev.toFixed(2) }}</strong></div>
                <div><span>Banquet Revenue Generated:</span><strong>RM{{ tableTotals.banquetRev.toFixed(2) }}</strong></div>
                <div><span>Other Revenue Generated:</span><strong>RM{{ tableTotals.otherRev.toFixed(2) }}</strong></div>
                <div><span>Total Revenue Generated:</span><strong>RM{{ tableTotals.confirm.toFixed(2) }}</strong></div>
              </div>
            </div>

            <!-- CARDS VIEW -->
            <div id="salesCardsView" class="sales-cards" :hidden="salesView !== 'cards' && !isPrintingReport">
              <p v-if="filteredSales.length === 0">No sales records found matching the criteria.</p>
              <article
                v-for="(s, idx) in paginatedSales"
                :key="s.id"
                class="sale-card"
                :class="{
                  'sale-card-room': String(s.source_id) === '1',
                  'sale-card-banquet': String(s.source_id) === '2',
                  'sale-card-other': String(s.source_id) !== '1' && String(s.source_id) !== '2'
                }"
              >
                <div class="sale-card-header">
                  <div class="sale-card-identity">
                    <div class="sale-card-company">{{ (currentSalesPage - 1) * salesPageSize + idx + 1 }}. {{ s.company_name }}</div>
                    <div class="sale-card-event" v-if="s.event_type">{{ s.event_type }}</div>
                  </div>
                  <div class="sale-card-date-block">
                    <span class="sale-card-date-value">{{ formatSaleDateRange(s.date_in, s.date_out) }}</span>
                  </div>
                </div>
                <div class="sale-card-highlights">
                  <div class="sale-card-highlight"><span class="sale-card-capacity-label">Guest pax</span><strong class="sale-card-capacity-value">{{ s.pax || 0 }}</strong></div>
                  <div class="sale-card-highlight"><span class="sale-card-capacity-label">Rooms</span><strong class="sale-card-capacity-value">{{ s.rooms_count || 0 }}</strong></div>
                  <div class="sale-card-highlight"><span class="sale-card-capacity-label">Rate quoted</span><strong class="sale-card-capacity-value">RM{{ Number(s.rate_quoted || 0).toFixed(2) }}</strong></div>
                </div>
                <div class="sale-card-confirm">
                  <span class="sale-card-confirm-label">Total Confirm</span>
                  <strong class="sale-card-revenue">RM{{ Number(s.revenue || 0).toFixed(2) }}</strong>
                </div>
                <dl class="sale-card-details">
                  <div><dt>Salesperson</dt><dd>{{ s.salesperson }}</dd></div>
                  <div><dt>Source</dt><dd>{{ s.source }}</dd></div>
                  <div><dt>A/C Type</dt><dd>{{ s.account_type }}</dd></div>
                  <div><dt>Status</dt><dd>{{ s.company_type }}</dd></div>
                </dl>
                <div class="sale-card-actions">
                  <button type="button" class="btn-edit" @click="populateEditForm(s.id)">Edit</button>
                  <button type="button" class="btn-delete" @click="requestSaleRemoval(s.id)">Delete</button>
                </div>
              </article>
            </div>

            <!-- PAGINATION -->
            <nav class="pagination">
              <span class="pagination-status">{{ paginationStatusText }}</span>
              <div class="pagination-actions">
                <button type="button" class="pagination-button" :disabled="currentSalesPage === 1" @click="changeSalesPage(-1)">Previous</button>
                <button type="button" class="pagination-button" :disabled="currentSalesPage === totalPages" @click="changeSalesPage(1)">Next</button>
              </div>
            </nav>
          </div>
        </section>
      </div>

      <!-- DASHBOARD PANEL -->
      <section id="dashboardPage" class="dashboard-section">
        <div class="card">
          <h2>Analytics</h2>
          <section class="analytics-card monthly-summary" aria-labelledby="monthlySummaryHeading">
            <div class="monthly-summary-header">
              <h3 id="monthlySummaryHeading">Monthly Summary</h3>
              <div class="monthly-summary-controls">
                <div class="monthly-summary-filter">
                  <label>Salesperson</label>
                  <select v-model="dashboardSalesperson">
                    <option value="">All Staff</option>
                    <option v-for="name in salespersonFilterOptions" :key="name" :value="name">{{ name }}</option>
                  </select>
                </div>
                <div class="monthly-summary-filter">
                  <label>Month</label>
                  <select v-model="currentDashboardMonth">
                    <option v-for="m in availableMonths" :key="m.value" :value="m.value">{{ m.name }}</option>
                  </select>
                </div>
              </div>
            </div>
            <div class="monthly-metrics">
              <div class="metric-card">
                <div class="metric-card-label">Revenue</div>
                <div class="metric-card-value">RM{{ monthlyRevenue.toFixed(2) }}</div>
                <div class="metric-card-detail">{{ dashboardPeriodLabel }}</div>
              </div>
              <div class="metric-card">  
                <div class="metric-card-label">Transactions</div>
                <div class="metric-card-value">{{ selectedMonthSales.length }}</div>
                <div class="metric-card-detail">{{ dashboardPeriodLabel }} · Date In</div>
              </div>
              <div class="metric-card">
                <div class="metric-card-label">Top Client Account</div>
                <div class="metric-card-value">{{ topClient.name }}</div>
                <div class="metric-card-detail">RM{{ topClient.rev.toFixed(2) }}</div>
              </div>
            </div>
          </section>

          <div class="dashboard-charts">
            <section class="analytics-card yearly-chart" aria-labelledby="performanceHeading">
              <div class="yearly-chart-header">
                <h3 id="performanceHeading" style="font-size: 17px;">Monthly Comparison</h3>
                <div class="yearly-chart-year">
                  <label>Year</label>
                  <select v-model="currentDashboardYear">
                    <option v-for="y in availableYears" :key="y" :value="y">{{ y }}</option>
                  </select>
                </div>
              </div>
              <div id="chartContainer" class="monthly-line-chart">
                <svg
                  class="monthly-line-svg"
                  viewBox="0 0 720 220"
                  role="group"
                  :aria-label="`Monthly revenue comparison for ${currentDashboardYear}`"
                  preserveAspectRatio="xMidYMid meet"
                >
                  <line
                    class="monthly-line-baseline"
                    x1="0"
                    :y1="monthlyLineChart.baseline"
                    x2="720"
                    :y2="monthlyLineChart.baseline"
                  />
                  <polyline
                    class="monthly-line"
                    :points="monthlyLineChart.line"
                  />
                  <g
                    v-for="point in monthlyLineChart.points"
                    :key="point.month"
                    class="monthly-line-point"
                    tabindex="0"
                    role="img"
                    :aria-label="`${point.month} ${currentDashboardYear}: RM${point.revenue.toFixed(2)}, ${point.transactions} transactions`"
                    @mouseenter="hoveredBarMonth = point.month"
                    @mouseleave="hoveredBarMonth = ''"
                    @focus="hoveredBarMonth = point.month"
                    @blur="hoveredBarMonth = ''"
                  >
                    <circle :cx="point.x" :cy="point.y" r="5" />
                    <text :x="point.x" y="210" text-anchor="middle">{{ point.month }}</text>
                    <foreignObject
                      v-if="hoveredBarMonth === point.month"
                      class="monthly-line-tooltip"
                      :x="Math.max(0, Math.min(point.x - 58, 604))"
                      :y="Math.max(0, point.y - 58)"
                      width="116"
                      height="50"
                      aria-hidden="true"
                    >
                      <div class="chart-hover-tooltip">
                        <strong>{{ point.month }} {{ currentDashboardYear }}</strong>
                        <span>Revenue: RM{{ point.revenue.toFixed(2) }}</span>
                        <span>Transactions: {{ point.transactions }}</span>
                      </div>
                    </foreignObject>
                  </g>
                </svg>
              </div>
            </section>

            <section class="analytics-card" aria-labelledby="categoryHeading">
              <h3 id="categoryHeading" style="margin-top: 0; font-size: 17px;">
                Category Distribution · {{ dashboardPeriodLabel }}
              </h3>
              <div id="monthPieChart" style="min-height: 180px;">
                <div v-if="categoryRevenue.length === 0" style="color:var(--palette-navy); font-size:12px;">
                  No sales data for pie chart
                </div>
                <div v-else class="pie-chart-layout">
                  <div class="pie-chart-wrap">
                    <div
                      v-if="hoveredCategoryData"
                      class="chart-hover-tooltip pie-chart-tooltip"
                      aria-hidden="true"
                    >
                      <strong>{{ hoveredCategoryData.category }}</strong>
                      <span>Revenue: RM{{ hoveredCategoryData.revenue.toFixed(2) }}</span>
                      <span>Share: {{ hoveredCategoryData.percentage.toFixed(1) }}%</span>
                    </div>
                    <svg
                      class="pie-chart"
                      viewBox="0 0 170 170"
                      role="group"
                      :aria-label="categoryChartLabel"
                    >
                      <template v-for="entry in categoryChartSlices" :key="entry.category">
                        <circle
                          v-if="entry.fullCircle"
                          cx="85"
                          cy="85"
                          r="85"
                          :fill="entry.color"
                          tabindex="0"
                          :aria-label="`${entry.category}: RM${entry.revenue.toFixed(2)}, ${entry.percentage.toFixed(1)}%`"
                          @mouseenter="hoveredCategory = entry.category"
                          @mouseleave="hoveredCategory = ''"
                          @focus="hoveredCategory = entry.category"
                          @blur="hoveredCategory = ''"
                        >
                          <title>{{ entry.category }}: RM{{ entry.revenue.toFixed(2) }} ({{ entry.percentage.toFixed(1) }}%)</title>
                        </circle>
                        <path
                          v-else
                          :d="entry.path"
                          :fill="entry.color"
                          tabindex="0"
                          :aria-label="`${entry.category}: RM${entry.revenue.toFixed(2)}, ${entry.percentage.toFixed(1)}%`"
                          @mouseenter="hoveredCategory = entry.category"
                          @mouseleave="hoveredCategory = ''"
                          @focus="hoveredCategory = entry.category"
                          @blur="hoveredCategory = ''"
                        >
                          <title>{{ entry.category }}: RM{{ entry.revenue.toFixed(2) }} ({{ entry.percentage.toFixed(1) }}%)</title>
                        </path>
                      </template>
                    </svg>
                  </div>
                  <div class="pie-chart-legend">
                    <div v-for="entry in categoryRevenue" :key="entry.category" class="pie-chart-legend-item">
                      <span class="pie-chart-legend-label">
                        <span class="pie-chart-swatch" :style="{ backgroundColor: entry.color }"></span>
                        <span>{{ entry.category }}</span>
                      </span>
                      <strong>RM{{ entry.revenue.toFixed(2) }} ({{ entry.percentage.toFixed(1) }}%)</strong>
                    </div>
                  </div>
                </div>
              </div>
              <div class="account-type-summary" aria-labelledby="accountTypeHeading">
                <h4 id="accountTypeHeading">A/C Type Summary</h4>
                <div class="account-type-counts">
                  <div class="account-type-count">
                    <strong>{{ accountTypeCounts.existing }}</strong><span>Existing</span>
                  </div>
                  <div class="account-type-count">
                    <strong>{{ accountTypeCounts.new }}</strong><span>New</span>
                  </div>
                  <div class="account-type-count">
                    <strong>{{ accountTypeCounts.growth }}</strong><span>Growth</span>
                  </div>
                </div>
              </div>
            </section>
          </div>
        </div>
      </section>
    </div>
  </div>
  <Teleport to="body">
    <div
      v-if="pendingSalespersonRemoval || pendingSaleRemoval"
      class="confirmation-modal-backdrop"
      @click.self="pendingSaleRemoval ? cancelSaleRemoval() : cancelSalespersonRemoval()"
      @keydown.esc="pendingSaleRemoval ? cancelSaleRemoval() : cancelSalespersonRemoval()"
    >
      <section
        class="confirmation-modal"
        role="alertdialog"
        aria-modal="true"
        :aria-labelledby="pendingSaleRemoval ? 'delete-sale-title' : 'remove-salesperson-title'"
        @keydown.esc.stop="pendingSaleRemoval ? cancelSaleRemoval() : cancelSalespersonRemoval()"
      >
        <h2 v-if="pendingSaleRemoval" id="delete-sale-title">Delete {{ pendingSaleRemoval.company_name }}?</h2>
        <h2 v-else id="remove-salesperson-title">Remove {{ pendingSalespersonRemoval.name }}?</h2>
        <div class="confirmation-modal-actions">
          <button
            ref="removeSalespersonCancelButton"
            type="button"
            class="confirmation-cancel"
            @click="pendingSaleRemoval ? cancelSaleRemoval() : cancelSalespersonRemoval()"
          >Cancel</button>
          <button v-if="pendingSaleRemoval" type="button" class="confirmation-remove" @click="deleteSale">Delete</button>
          <button v-else type="button" class="confirmation-remove" @click="removeSalesperson">Remove</button>
        </div>
      </section>
    </div>
  </Teleport>
</template>
