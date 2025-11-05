<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { Bar, Line } from 'vue-chartjs'
import {
  Chart as ChartJS,
  Title,
  Tooltip,
  Legend,
  BarElement,
  LineElement,
  PointElement,
  CategoryScale,
  LinearScale,
} from 'chart.js'
import Papa from 'papaparse'

ChartJS.register(Title, Tooltip, Legend, BarElement, LineElement, PointElement, CategoryScale, LinearScale)

function fmtNumber(n) {
  const num = Number(n)
  return Number.isFinite(num) ? num.toLocaleString('tr-TR', { maximumFractionDigits: 2 }) : '—'
}
function fmtPrice(n) {
  const num = Number(n)
  return Number.isFinite(num) ? num.toLocaleString('tr-TR', { minimumFractionDigits: 4, maximumFractionDigits: 4 }) : '—'
}

const rawRows = ref([])
const selectedItem = ref('')
const availableItems = ref([])
const chartBoxEl = ref(null)
const theme = ref('light') // 'dark' | 'light'
const volumeColor = ref('#42b883')
const priceColor = ref('#646cff')
const displayMode = ref('split') // 'combined' | 'split'

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
          item: String(r['Item'] ?? '').trim(),
          amount: Number(r['Amount'] ?? NaN),
          money: Number(r['Money'] ?? NaN),
          dateIso: String(r['Date_UTC_ISO'] ?? r['Date'] ?? '').trim()
        }))
        .filter((r) => r.item && Number.isFinite(r.amount) && Number.isFinite(r.money) && r.dateIso)

      rawRows.value = cleaned

      const items = Array.from(new Set(cleaned.map((r) => r.item))).sort()
      availableItems.value = items
      if (!items.includes(selectedItem.value)) selectedItem.value = items[0] ?? ''
    }
  })
}

const filtered = computed(() => rawRows.value.filter((r) => !selectedItem.value || r.item === selectedItem.value))

function toBucketKey(iso, stepHours) {
  const d = new Date(iso)
  const bucketHour = Math.floor(d.getUTCHours() / stepHours) * stepHours
  const y = d.getUTCFullYear()
  const m = String(d.getUTCMonth() + 1).padStart(2, '0')
  const day = String(d.getUTCDate()).padStart(2, '0')
  const h = String(bucketHour).padStart(2, '0')
  if (stepHours === 1) return `${y}-${m}-${day} ${h}:00 UTC`
  const endH = String((bucketHour + stepHours - 1) % 24).padStart(2, '0')
  return `${y}-${m}-${day} ${h}:00-${endH}:59 UTC`
}

function aggregate(rows, stepHours) {
  const map = new Map()
  for (const r of rows) {
    const key = toBucketKey(r.dateIso, stepHours)
    if (!map.has(key)) map.set(key, { totalAmount: 0, totalMoney: 0, firstIso: r.dateIso })
    const b = map.get(key)
    b.totalAmount += r.amount
    b.totalMoney += r.money
    if (new Date(r.dateIso) < new Date(b.firstIso)) b.firstIso = r.dateIso
  }
  const entries = Array.from(map.entries())
  // Sırayı stabil kılmak için bucket başlangıç zamanına göre sırala
  entries.sort((a, b) => new Date(a[1].firstIso) - new Date(b[1].firstIso))
  return entries
}

const hourlyBuckets = computed(() => {
  const perHour = aggregate(filtered.value, 1)
  if (perHour.length > 300) return aggregate(filtered.value, 24)
  if (perHour.length > 30) return aggregate(filtered.value, 4)
  return perHour
})

const currentGrouping = computed(() => {
  const perHour = aggregate(filtered.value, 1)
  if (perHour.length > 300) return '24h'
  if (perHour.length > 30) return '4h'
  return '1h'
})

function toDayLabel(iso) {
  const d = new Date(iso)
  const months = ['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec']
  const m = months[d.getUTCMonth()]
  const day = d.getUTCDate()
  return `${m} ${day}`
}

const labels = computed(() => hourlyBuckets.value.map(([, v]) => toDayLabel(v.firstIso)))
const totalVolumes = computed(() => hourlyBuckets.value.map(([, v]) => v.totalAmount))
const avgPrices = computed(() => hourlyBuckets.value.map(([, v]) => v.totalAmount > 0 ? v.totalMoney / v.totalAmount : 0))

// Overall averages for right-side summary panel
const overallAveragePrice = computed(() => {
  const rows = filtered.value
  let money = 0
  let amount = 0
  for (const r of rows) {
    money += r.money
    amount += r.amount
  }
  return amount > 0 ? money / amount : 0
})

function groupByUtcDay(rows) {
  const map = new Map()
  for (const r of rows) {
    const d = new Date(r.dateIso)
    const key = `${d.getUTCFullYear()}-${String(d.getUTCMonth() + 1).padStart(2, '0')}-${String(d.getUTCDate()).padStart(2, '0')}`
    if (!map.has(key)) map.set(key, 0)
    map.set(key, map.get(key) + r.amount)
  }
  return Array.from(map.values())
}

function groupByUtcMonth(rows) {
  const map = new Map()
  for (const r of rows) {
    const d = new Date(r.dateIso)
    const key = `${d.getUTCFullYear()}-${String(d.getUTCMonth() + 1).padStart(2, '0')}`
    if (!map.has(key)) map.set(key, 0)
    map.set(key, map.get(key) + r.amount)
  }
  return Array.from(map.values())
}

const averageVolumePerBucket = computed(() => {
  const values = hourlyBuckets.value.map(([, v]) => v.totalAmount)
  if (!values.length) return 0
  return values.reduce((a, b) => a + b, 0) / values.length
})

const averageDailyVolume = computed(() => {
  const dailyTotals = groupByUtcDay(filtered.value)
  if (!dailyTotals.length) return 0
  return dailyTotals.reduce((a, b) => a + b, 0) / dailyTotals.length
})

const averageVolumeLabel = computed(() => {
  if (currentGrouping.value === '24h') return 'Avg Volume (day)'
  if (currentGrouping.value === '4h') return 'Avg Volume (per 4h)'
  return 'Avg Volume (per 1h)'
})

const averageVolumeDisplay = computed(() => {
  if (currentGrouping.value === '24h') return averageDailyVolume.value
  return averageVolumePerBucket.value
})

function hexWithAlpha(hex, alpha) {
  const m = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex)
  if (!m) return hex
  const r = parseInt(m[1], 16)
  const g = parseInt(m[2], 16)
  const b = parseInt(m[3], 16)
  return `rgba(${r}, ${g}, ${b}, ${alpha})`
}

const chartData = computed(() => {
  const vol = volumeColor.value
  const price = priceColor.value
  return {
    labels: labels.value,
    datasets: [
      {
        type: 'bar',
        label: 'Total Volume',
        data: totalVolumes.value,
        yAxisID: 'y',
        backgroundColor: hexWithAlpha(vol, 0.75),
        borderColor: vol,
        borderWidth: 1,
        borderRadius: 6,
        hoverBackgroundColor: hexWithAlpha(vol, 0.95)
      },
      {
        type: 'line',
        label: 'Average Price',
        data: avgPrices.value,
        yAxisID: 'y1',
        borderColor: price,
        backgroundColor: price,
        borderDash: [6, 4],
        borderWidth: 2,
        pointRadius: 5,
        pointHoverRadius: 8,
        tension: 0.25,
        fill: false,
        order: 3
      }
    ]
  }
})

// Split charts (top: price line, bottom: volume bars)
const priceChartData = computed(() => {
  const price = priceColor.value
  return {
    labels: labels.value,
    datasets: [
      {
        type: 'line',
        label: 'Average Price',
        data: avgPrices.value,
        borderColor: price,
        backgroundColor: price,
        borderDash: [6, 4],
        borderWidth: 2,
        pointRadius: 5,
        pointHoverRadius: 8,
        tension: 0.25,
        fill: false
      }
    ]
  }
})

const volumeChartData = computed(() => {
  const vol = volumeColor.value
  return {
    labels: labels.value,
    datasets: [
      {
        label: 'Total Volume',
        data: totalVolumes.value,
        backgroundColor: hexWithAlpha(vol, 0.75),
        borderColor: vol,
        borderWidth: 1,
        borderRadius: 6,
        hoverBackgroundColor: hexWithAlpha(vol, 0.95)
      }
    ]
  }
})

const chartOptions = computed(() => {
  const isDark = theme.value === 'dark'
  const grid = isDark ? 'rgba(255,255,255,0.06)' : 'rgba(0,0,0,0.08)'
  const labelColor = isDark ? '#a9abb3' : '#333'
  const titleColor = isDark ? '#eaeaf0' : '#111'
  const ttBg = isDark ? 'rgba(20,20,24,0.95)' : 'rgba(255,255,255,0.97)'
  const ttBorder = isDark ? '#2a2a2e' : '#ddd'
  const ttText = isDark ? '#0' : '#0' // bodyColor/titleColor aşağıda set ediliyor

  return {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      legend: { position: 'top', labels: { color: labelColor } },
      title: { display: false, text: 'Hourly Total Volume and Average Price', color: titleColor, position: 'top', font: { size: 14, weight: 'bold' }, padding: { top: 8, bottom: 12 } } ,
      tooltip: {
        backgroundColor: ttBg,
        borderColor: ttBorder,
        borderWidth: 1,
        titleColor: isDark ? '#0ff' : '#111',
        bodyColor: isDark ? '#eaeaf0' : '#111',
        callbacks: {
          title: (items) => items?.[0]?.label ? `${items[0].label}` : '',
          label: (ctx) => {
            const val = ctx.parsed?.y
            const label = ctx.dataset.label
            if (label === 'Average Price') return `${label}: ${fmtPrice(val)}`
            return `${label}: ${fmtNumber(val)}`
          }
        }
      }
    },
    scales: {
      x: { stacked: false, ticks: { color: labelColor }, grid: { color: grid } },
      y: { position: 'left', title: { display: true, text: 'Value', color: labelColor }, ticks: { color: labelColor }, grid: { color: grid } }
    }
  }
})

const priceChartOptions = computed(() => {
  const base = chartOptions.value
  return {
    ...base,
    plugins: { ...base.plugins, title: { display: false, text: 'Average Price', color: base.plugins.title.color, position: 'top', font: { size: 14, weight: 'bold' }, padding: { top: 4, bottom: 8 } } },
    scales: {
      x: base.scales.x,
      y: { ...base.scales.y, title: { display: true, text: 'Price', color: base.scales.y.title.color }, ticks: { ...base.scales.y.ticks, callback: (v) => fmtPrice(v) } }
    }
  }
})

const volumeChartOptions = computed(() => {
  const base = chartOptions.value
  return {
    ...base,
    plugins: { ...base.plugins, title: { display: false, text: 'Total Volume', color: base.plugins.title.color, position: 'top', font: { size: 14, weight: 'bold' }, padding: { top: 4, bottom: 8 } } },
    scales: {
      x: base.scales.x,
      y: { ...base.scales.y, title: { display: true, text: 'Volume', color: base.scales.y.title.color }, ticks: base.scales.y.ticks }
    }
  }
})

// tam ekran özelliği kaldırıldı
</script>

<template>
  <div class="wrapper">
    <div class="controls">
      <div class="file-label">
        <input type="file" accept=".csv,text/csv" @change="onFileChange" />
      </div>

      <div class="select-label" v-if="availableItems.length > 1">
        <select v-model="selectedItem">
          <option v-for="it in availableItems" :key="it" :value="it">{{ it }}</option>
        </select>
      </div>
      
      <button class="theme-btn" type="button" @click="theme = theme==='dark' ? 'light' : 'dark'">{{ theme==='dark' ? '☀️' : '🌙' }}</button>
      <input class="color" type="color" v-model="volumeColor" title="Hacim Rengi" />
      <input class="color" type="color" v-model="priceColor" title="Ortalama Fiyat Rengi" />
      <button class="mode-btn" :class="{active: displayMode==='combined'}" type="button" @click="displayMode='combined'">Birleşik</button>
      <button class="mode-btn" :class="{active: displayMode==='split'}" type="button" @click="displayMode='split'">Bölünmüş</button>
    </div>

    <div class="chart-box" :class="theme==='dark' ? 'dark' : 'light'" ref="chartBoxEl">
    <template v-if="displayMode==='combined'">
      <Bar :data="chartData" :options="chartOptions" />
      <div class="stats-panel">
        <div class="stat">
          <div class="stat-label">Average Price</div>
          <div class="stat-value">{{ fmtPrice(overallAveragePrice) }}</div>
        </div>
        <div class="divider"></div>
        <div class="stat">
          <div class="stat-label">{{ averageVolumeLabel }}</div>
          <div class="stat-value">{{ fmtNumber(averageVolumeDisplay) }}</div>
        </div>
      </div>
    </template>
    <div v-else class="split">
        <div class="pane price">
          <Line :data="priceChartData" :options="priceChartOptions" />
        <div class="stats-panel">
          <div class="stat">
            <div class="stat-label">Average Price</div>
            <div class="stat-value">{{ fmtPrice(overallAveragePrice) }}</div>
          </div>
        </div>
        </div>
        <div class="pane volume">
          <Bar :data="volumeChartData" :options="volumeChartOptions" />
        <div class="stats-panel">
          <div class="stat">
            <div class="stat-label">{{ averageVolumeLabel }}</div>
            <div class="stat-value">{{ fmtNumber(averageVolumeDisplay) }}</div>
          </div>
        </div>
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
  align-items: center;
}
.file-label, .select-label { display: flex; flex-direction: column; gap: 4px; font-weight: 600; }
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
  margin-left: -17px; /* az sola kaydır */
}
.chart-box.light {
  background: #ffffff;
  border: 1px solid #dcdcdc;
}
.stats-panel {
  position: absolute;
  top: 6px;
  right: 6px;
  display: flex;
  flex-direction: column;
  gap: 4px;
  min-width: 120px;
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
.stat { display: flex; flex-direction: column; }
.stat-label { font-size: 11px; opacity: 0.9; }
.stat-value { font-size: 13px; font-weight: 700; }
.divider { height: 1px; background: rgba(0,0,0,0.08); }
.chart-box.dark .divider { background: rgba(255,255,255,0.08); }
.split {
  display: grid;
  grid-template-rows: 1fr 1fr;
  gap: 8px;
  height: 100%;
}
.pane { position: relative; }
input[type="file"], select {
  padding: 3px 6px;
  background: #1a1a1e;
  border: 1px solid #2a2a2e;
  border-radius: 6px;
  color: #eaeaf0;
  font-size: 12px;
}
/* tam ekran butonu kaldırıldı */
.theme-btn {
  padding: 3px 6px;
  background: #1a1a1e;
  border: 1px solid #2a2a2e;
  color: #eaeaf0;
  border-radius: 6px;
  cursor: pointer;
  font-size: 12px;
}
/* tam ekran çıkış butonu kaldırıldı */

.color { width: 28px; height: 28px; padding: 0; background: transparent; border: 1px solid #2a2a2e; border-radius: 4px; }

.mode-btn { padding: 3px 6px; background: #1a1a1e; border: 1px solid #2a2a2e; color: #eaeaf0; border-radius: 6px; cursor: pointer; font-size: 12px; }
.mode-btn.active { background: #2a2a2e; }

</style>

<style scoped>
/* Responsive yerleşimler */
@media (min-width: 1440px) { .chart-box { height: clamp(520px, 66vh, 820px); } }

@media (max-width: 1024px) {
  .chart-box { height: clamp(340px, 56vh, 600px); }
  .controls { top: 12px; left: 12px; gap: 6px; }
}

@media (max-width: 768px) {
  .wrapper { padding: 8px; }
  .chart-box { height: clamp(320px, 54vh, 540px); margin-left: -8px; }
  .controls { top: auto; bottom: 12px; left: 12px; right: auto; gap: 6px; flex-wrap: wrap; }
  input[type="file"], select { font-size: 11px; padding: 4px 6px; }
  .theme-btn, .mode-btn { font-size: 11px; padding: 3px 6px; }
  .color { width: 26px; height: 26px; }
  .stats-panel { top: 6px; right: 6px; min-width: 110px; padding: 6px 8px; gap: 4px; }
  .stat-value { font-size: 12px; }
}

@media (max-width: 480px) {
  .chart-box { height: clamp(300px, 52vh, 480px); margin-left: -6px; }
  .controls { bottom: 8px; left: 8px; right: auto; gap: 4px; }
  input[type="file"], select { font-size: 10px; padding: 3px 5px; }
  .theme-btn, .mode-btn { font-size: 10px; padding: 3px 5px; }
  .color { width: 24px; height: 24px; }
  .stats-panel { min-width: 100px; }
}
</style>


