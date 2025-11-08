<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { Bar } from 'vue-chartjs'
import {
  Chart as ChartJS,
  Title,
  Tooltip,
  Legend,
  BarElement,
  CategoryScale,
  LinearScale,
} from 'chart.js'
import Papa from 'papaparse'

ChartJS.register(Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale)

function fmtNumber(n) {
  const num = Number(n)
  return Number.isFinite(num) ? num.toLocaleString('tr-TR', { maximumFractionDigits: 2 }) : '—'
}
function fmtMoney(n) {
  const num = Number(n)
  return Number.isFinite(num) ? num.toLocaleString('tr-TR', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) : '—'
}

const rawRows = ref([])
const chartBoxEl = ref(null)
const theme = ref('light') // 'dark' | 'light'
const barColor = ref('#42b883')
const topN = ref(20) // Show top N taxpayers
const buyerIdInput = ref('')
const allowedBuyerIds = ref([]) // Array of buyer IDs to filter
const showBuyerIdsList = ref(false) // Toggle to show/hide buyer IDs list

// Load buyer IDs from localStorage on mount
onMounted(() => {
  loadBuyerIds()
})

// Load buyer IDs from localStorage
function loadBuyerIds() {
  const stored = localStorage.getItem('taxPayersBuyerIds')
  if (stored) {
    try {
      allowedBuyerIds.value = JSON.parse(stored)
    } catch (e) {
      console.error('Failed to parse stored buyer IDs:', e)
    }
  }
}

// Watch for changes and save to localStorage
watch(allowedBuyerIds, (newVal) => {
  localStorage.setItem('taxPayersBuyerIds', JSON.stringify(newVal))
}, { deep: true })

// Add buyer ID
function addBuyerId() {
  const id = buyerIdInput.value.trim()
  if (id && !allowedBuyerIds.value.includes(id)) {
    allowedBuyerIds.value.push(id)
    buyerIdInput.value = ''
  }
}

// Remove buyer ID
function removeBuyerId(id) {
  const index = allowedBuyerIds.value.indexOf(id)
  if (index > -1) {
    allowedBuyerIds.value.splice(index, 1)
  }
}

function onFileChange(e) {
  const file = e.target.files?.[0]
  if (!file) return

  Papa.parse(file, {
    header: true,
    skipEmptyLines: true,
    complete: (result) => {
      const rows = Array.isArray(result.data) ? result.data : []
      const cleaned = rows
        .map((r) => ({
          buyerId: String(r['Buyer Id'] ?? r['Buyer Id'] ?? '').trim(),
          buyer: String(r['Buyer'] ?? '').trim(),
          money: Number(r['Money'] ?? NaN),
        }))
        .filter((r) => r.buyerId && r.buyer && Number.isFinite(r.money) && r.money > 0)

      rawRows.value = cleaned
    }
  })
}

// Group by Buyer Id and Buyer, sum total money (total wage)
// Only include buyers from allowedBuyerIds list
const buyerTotals = computed(() => {
  // If no buyer IDs specified, return empty
  if (allowedBuyerIds.value.length === 0) {
    return []
  }
  
  const map = new Map()
  const allowedSet = new Set(allowedBuyerIds.value)
  
  // Filter rows by allowed buyer IDs
  for (const r of rawRows.value) {
    // Only process if buyer ID is in allowed list
    if (!allowedSet.has(r.buyerId)) {
      continue
    }
    
    const key = `${r.buyerId}|${r.buyer}`
    if (!map.has(key)) {
      map.set(key, {
        buyerId: r.buyerId,
        buyer: r.buyer,
        totalWage: 0,
        taxPaid: 0
      })
    }
    const entry = map.get(key)
    entry.totalWage += r.money
  }
  
  // Calculate tax (10% of total wage)
  for (const entry of map.values()) {
    entry.taxPaid = entry.totalWage * 0.1
  }
  
  // Convert to array, filter out those with no tax paid, and sort by tax paid (descending)
  const entries = Array.from(map.values())
    .filter(entry => entry.taxPaid > 0) // Only show buyers who paid tax
    .sort((a, b) => b.taxPaid - a.taxPaid)
  
  return entries
})

// Top N taxpayers
const topTaxPayers = computed(() => {
  return buyerTotals.value.slice(0, topN.value)
})

// Chart data
const chartData = computed(() => {
  const top = topTaxPayers.value
  return {
    labels: top.map(t => t.buyer),
    datasets: [
      {
        label: 'Tax Paid',
        data: top.map(t => t.taxPaid),
        backgroundColor: barColor.value,
        borderColor: barColor.value,
        borderWidth: 1,
        borderRadius: 6,
      }
    ]
  }
})

// Overall statistics
const totalTaxPaid = computed(() => {
  return buyerTotals.value.reduce((sum, b) => sum + b.taxPaid, 0)
})

const totalWagePaid = computed(() => {
  return buyerTotals.value.reduce((sum, b) => sum + b.totalWage, 0)
})

const averageTaxPerBuyer = computed(() => {
  if (buyerTotals.value.length === 0) return 0
  return totalTaxPaid.value / buyerTotals.value.length
})

const chartOptions = computed(() => {
  const isDark = theme.value === 'dark'
  const grid = isDark ? 'rgba(255,255,255,0.06)' : 'rgba(0,0,0,0.08)'
  const labelColor = isDark ? '#a9abb3' : '#333'
  const titleColor = isDark ? '#eaeaf0' : '#111'
  const ttBg = isDark ? 'rgba(20,20,24,0.95)' : 'rgba(255,255,255,0.97)'
  const ttBorder = isDark ? '#2a2a2e' : '#ddd'

  return {
    responsive: true,
    maintainAspectRatio: false,
    indexAxis: 'y', // Horizontal bars
    plugins: {
      legend: { display: false },
      title: { display: false },
      tooltip: {
        backgroundColor: ttBg,
        borderColor: ttBorder,
        borderWidth: 1,
        titleColor: isDark ? '#0ff' : '#111',
        bodyColor: isDark ? '#eaeaf0' : '#111',
        callbacks: {
          title: (items) => items?.[0]?.label ? `${items[0].label}` : '',
          label: (ctx) => {
            const val = ctx.parsed?.x
            const buyer = topTaxPayers.value[ctx.dataIndex]
            if (!buyer) return `Tax: ${fmtMoney(val)}`
            return [
              `Tax Paid: ${fmtMoney(val)}`,
              `Total Wage: ${fmtMoney(buyer.totalWage)}`,
              `Buyer ID: ${buyer.buyerId}`
            ]
          }
        }
      }
    },
    scales: {
      x: {
        title: { display: true, text: 'Tax Paid', color: labelColor },
        ticks: { color: labelColor },
        grid: { color: grid }
      },
      y: {
        ticks: { color: labelColor },
        grid: { color: grid, display: false }
      }
    }
  }
})
</script>

<template>
  <div class="wrapper">
    <div class="controls">
      <div class="file-label">
        <input type="file" accept=".csv,text/csv" @change="onFileChange" />
      </div>
      
      <div class="buyer-ids-section">
        <div class="buyer-id-input-group">
          <input 
            type="text" 
            v-model="buyerIdInput" 
            placeholder="Buyer ID"
            class="buyer-id-input"
            @keyup.enter="addBuyerId"
          />
          <button class="add-btn" @click="addBuyerId" type="button">Add</button>
          <button class="view-btn" @click="showBuyerIdsList = !showBuyerIdsList" type="button">
            {{ showBuyerIdsList ? 'Hide' : 'View' }}
          </button>
        </div>
        <div v-if="showBuyerIdsList" class="buyer-ids-list">
          <div v-for="id in allowedBuyerIds" :key="id" class="buyer-id-tag">
            <span>{{ id }}</span>
            <button class="remove-btn" @click="removeBuyerId(id)" type="button">×</button>
          </div>
          <div v-if="allowedBuyerIds.length === 0" class="empty-buyer-ids">
            No buyer IDs added yet
          </div>
        </div>
      </div>
      
      <div class="input-group">
        <label>Top N:</label>
        <input type="number" v-model.number="topN" min="5" max="50" class="number-input" />
      </div>
      
      <button class="theme-btn" type="button" @click="theme = theme==='dark' ? 'light' : 'dark'">
        {{ theme==='dark' ? '☀️' : '🌙' }}
      </button>
      <input class="color" type="color" v-model="barColor" title="Bar Color" />
    </div>

    <div class="chart-box" :class="theme==='dark' ? 'dark' : 'light'" ref="chartBoxEl">
      <Bar v-if="topTaxPayers.length > 0" :data="chartData" :options="chartOptions" />
      <div v-else class="empty-state">
        <p v-if="allowedBuyerIds.length === 0">
          Please add Buyer IDs to filter data
        </p>
        <p v-else-if="rawRows.length === 0">
          Please upload a CSV file to view tax payer data
        </p>
        <p v-else>
          No tax data found for the specified Buyer IDs
        </p>
      </div>
      
      <div v-if="topTaxPayers.length > 0" class="stats-panel">
        <div class="stat">
          <div class="stat-label">Total Tax Paid</div>
          <div class="stat-value">{{ fmtMoney(totalTaxPaid) }}</div>
        </div>
        <div class="divider"></div>
        <div class="stat">
          <div class="stat-label">Total Wage Paid</div>
          <div class="stat-value">{{ fmtMoney(totalWagePaid) }}</div>
        </div>
        <div class="divider"></div>
        <div class="stat">
          <div class="stat-label">Avg Tax per Buyer</div>
          <div class="stat-value">{{ fmtMoney(averageTaxPerBuyer) }}</div>
        </div>
        <div class="divider"></div>
        <div class="stat">
          <div class="stat-label">Total Buyers</div>
          <div class="stat-value">{{ buyerTotals.length }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.wrapper {
  display: flex;
  flex-direction: column;
  gap: 16px;
  padding: 16px;
}
.controls {
  position: fixed;
  top: 12px;
  left: 12px;
  z-index: 1000;
  display: flex;
  flex-direction: row;
  gap: 8px;
  align-items: flex-start;
  flex-wrap: wrap;
  max-width: calc(100vw - 200px);
}
.file-label {
  display: flex;
  flex-direction: column;
  gap: 4px;
  font-weight: 600;
}
.buyer-ids-section {
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 8px;
  background: rgba(26, 26, 30, 0.8);
  border: 1px solid #2a2a2e;
  border-radius: 6px;
  min-width: 200px;
  max-width: 300px;
}
.buyer-id-input-group {
  display: flex;
  gap: 4px;
}
.buyer-id-input {
  flex: 1;
  padding: 4px 8px;
  background: #1a1a1e;
  border: 1px solid #2a2a2e;
  border-radius: 4px;
  color: #eaeaf0;
  font-size: 12px;
}
.buyer-id-input::placeholder {
  color: #666;
}
.add-btn {
  padding: 4px 10px;
  background: #42b883;
  border: none;
  border-radius: 4px;
  color: #fff;
  font-size: 12px;
  cursor: pointer;
  font-weight: 600;
}
.add-btn:hover {
  background: #35a06d;
}
.view-btn {
  padding: 4px 10px;
  background: #1a1a1e;
  border: 1px solid #2a2a2e;
  border-radius: 4px;
  color: #eaeaf0;
  font-size: 12px;
  cursor: pointer;
  font-weight: 600;
}
.view-btn:hover {
  background: #2a2a2e;
}
.buyer-ids-list {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  max-height: 100px;
  overflow-y: auto;
}
.buyer-id-tag {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 3px 6px;
  background: #2a2a2e;
  border-radius: 4px;
  font-size: 11px;
  color: #eaeaf0;
}
.remove-btn {
  background: transparent;
  border: none;
  color: #a9abb3;
  cursor: pointer;
  font-size: 16px;
  line-height: 1;
  padding: 0;
  width: 16px;
  height: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.remove-btn:hover {
  color: #ff6b6b;
}
.empty-buyer-ids {
  padding: 8px;
  text-align: center;
  color: #666;
  font-size: 11px;
  font-style: italic;
}
.input-group {
  display: flex;
  align-items: center;
  gap: 4px;
}
.input-group label {
  font-size: 12px;
  color: #eaeaf0;
}
.number-input {
  width: 50px;
  padding: 3px 6px;
  background: #1a1a1e;
  border: 1px solid #2a2a2e;
  border-radius: 6px;
  color: #eaeaf0;
  font-size: 12px;
}
.chart-box {
  position: relative;
  width: 100%;
  height: clamp(380px, 60vh, 680px);
  background: #101014;
  border: 1px solid #2a2a2e;
  border-radius: 8px;
  margin-top: 62px;
  padding: 8px;
  margin-right: 5px;
  margin-left: -17px;
}
.chart-box.light {
  background: #ffffff;
  border: 1px solid #dcdcdc;
}
.empty-state {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: #a9abb3;
  font-size: 14px;
}
.stats-panel {
  position: absolute;
  top: 6px;
  right: 6px;
  display: flex;
  flex-direction: column;
  gap: 4px;
  min-width: 160px;
  padding: 6px 8px;
  border-radius: 6px;
  background: rgba(255,255,255,0.65);
  color: #222;
  border: 1px solid rgba(0,0,0,0.06);
  box-shadow: 0 1px 6px rgba(0,0,0,0.06);
  backdrop-filter: saturate(150%) blur(6px);
  pointer-events: none;
}
.chart-box.dark .stats-panel {
  background: rgba(18,18,22,0.55);
  color: #eaeaf0;
  border: 1px solid #2a2a2e;
  box-shadow: 0 1px 8px rgba(0,0,0,0.35);
}
.stat {
  display: flex;
  flex-direction: column;
}
.stat-label {
  font-size: 11px;
  opacity: 0.9;
}
.stat-value {
  font-size: 13px;
  font-weight: 700;
}
.divider {
  height: 1px;
  background: rgba(0,0,0,0.08);
  margin: 2px 0;
}
.chart-box.dark .divider {
  background: rgba(255,255,255,0.08);
}
input[type="file"] {
  padding: 3px 6px;
  background: #1a1a1e;
  border: 1px solid #2a2a2e;
  border-radius: 6px;
  color: #eaeaf0;
  font-size: 12px;
}
.theme-btn {
  padding: 3px 6px;
  background: #1a1a1e;
  border: 1px solid #2a2a2e;
  color: #eaeaf0;
  border-radius: 6px;
  cursor: pointer;
  font-size: 12px;
}
.color {
  width: 28px;
  height: 28px;
  padding: 0;
  background: transparent;
  border: 1px solid #2a2a2e;
  border-radius: 4px;
}
</style>

<style scoped>
/* Responsive yerleşimler */
@media (min-width: 1440px) {
  .chart-box {
    height: clamp(520px, 66vh, 820px);
  }
}

@media (max-width: 1024px) {
  .chart-box {
    height: clamp(340px, 56vh, 600px);
  }
  .controls {
    top: 12px;
    left: 12px;
    gap: 6px;
  }
}

@media (max-width: 768px) {
  .wrapper {
    padding: 8px;
  }
  .chart-box {
    height: clamp(320px, 54vh, 540px);
    margin-left: -8px;
  }
  .controls {
    top: auto;
    bottom: 12px;
    left: 12px;
    right: auto;
    gap: 6px;
    flex-wrap: wrap;
  }
  input[type="file"], .number-input {
    font-size: 11px;
    padding: 4px 6px;
  }
  .theme-btn {
    font-size: 11px;
    padding: 3px 6px;
  }
  .color {
    width: 26px;
    height: 26px;
  }
  .stats-panel {
    top: 6px;
    right: 6px;
    min-width: 140px;
    padding: 6px 8px;
    gap: 4px;
  }
  .stat-value {
    font-size: 12px;
  }
}

@media (max-width: 480px) {
  .chart-box {
    height: clamp(300px, 52vh, 480px);
    margin-left: -6px;
  }
  .controls {
    bottom: 8px;
    left: 8px;
    right: auto;
    gap: 4px;
  }
  input[type="file"], .number-input {
    font-size: 10px;
    padding: 3px 5px;
  }
  .theme-btn {
    font-size: 10px;
    padding: 3px 5px;
  }
  .color {
    width: 24px;
    height: 24px;
  }
  .stats-panel {
    min-width: 120px;
  }
}
</style>

