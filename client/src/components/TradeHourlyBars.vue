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
const isFullscreen = ref(false)
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

function toDayLabel(iso) {
  const d = new Date(iso)
  return String(d.getUTCDate()).padStart(2, '0')
}

const labels = computed(() => hourlyBuckets.value.map(([, v]) => toDayLabel(v.firstIso)))
const totalVolumes = computed(() => hourlyBuckets.value.map(([, v]) => v.totalAmount))
const avgPrices = computed(() => hourlyBuckets.value.map(([, v]) => v.totalAmount > 0 ? v.totalMoney / v.totalAmount : 0))

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
        label: 'Toplam Hacim',
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
        label: 'Ortalama Fiyat',
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
        label: 'Ortalama Fiyat',
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
        label: 'Toplam Hacim',
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
      title: { display: true, text: 'Saatlik Toplam Hacim ve Ortalama Fiyat', color: titleColor } ,
      tooltip: {
        backgroundColor: ttBg,
        borderColor: ttBorder,
        borderWidth: 1,
        titleColor: isDark ? '#0ff' : '#111',
        bodyColor: isDark ? '#eaeaf0' : '#111',
        callbacks: {
          title: (items) => items?.[0]?.label ? `Gün: ${items[0].label}` : '',
          label: (ctx) => {
            const val = ctx.parsed?.y
            const label = ctx.dataset.label
            if (label === 'Ortalama Fiyat') return `${label}: ${fmtPrice(val)}`
            return `${label}: ${fmtNumber(val)}`
          }
        }
      }
    },
    scales: {
      x: { stacked: false, ticks: { color: labelColor }, grid: { color: grid } },
      y: { position: 'left', title: { display: true, text: 'Değer', color: labelColor }, ticks: { color: labelColor }, grid: { color: grid } }
    }
  }
})

const priceChartOptions = computed(() => {
  const base = chartOptions.value
  return {
    ...base,
    plugins: { ...base.plugins, title: { display: true, text: 'Ortalama Fiyat', color: base.plugins.title.color } },
    scales: {
      x: base.scales.x,
      y: { ...base.scales.y, title: { display: true, text: 'Fiyat', color: base.scales.y.title.color }, ticks: { ...base.scales.y.ticks, callback: (v) => fmtPrice(v) } }
    }
  }
})

const volumeChartOptions = computed(() => {
  const base = chartOptions.value
  return {
    ...base,
    plugins: { ...base.plugins, title: { display: true, text: 'Toplam Hacim', color: base.plugins.title.color } },
    scales: {
      x: base.scales.x,
      y: { ...base.scales.y, title: { display: true, text: 'Hacim', color: base.scales.y.title.color }, ticks: base.scales.y.ticks }
    }
  }
})

function toggleFullscreen() {
  const el = chartBoxEl.value
  if (!el) return
  if (!document.fullscreenElement) {
    if (el.requestFullscreen) el.requestFullscreen()
  } else {
    if (document.exitFullscreen) document.exitFullscreen()
  }
}

function onFsChange() {
  isFullscreen.value = !!document.fullscreenElement
}

onMounted(() => {
  document.addEventListener('fullscreenchange', onFsChange)
})

onUnmounted(() => {
  document.removeEventListener('fullscreenchange', onFsChange)
})
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
      <button class="fs-btn" type="button" @click="toggleFullscreen">⤢ Tam Ekran</button>
      <button class="theme-btn" type="button" @click="theme = theme==='dark' ? 'light' : 'dark'">{{ theme==='dark' ? '☀️' : '🌙' }}</button>
      <input class="color" type="color" v-model="volumeColor" title="Hacim Rengi" />
      <input class="color" type="color" v-model="priceColor" title="Ortalama Fiyat Rengi" />
      <button class="mode-btn" :class="{active: displayMode==='combined'}" type="button" @click="displayMode='combined'">Birleşik</button>
      <button class="mode-btn" :class="{active: displayMode==='split'}" type="button" @click="displayMode='split'">Bölünmüş</button>
    </div>

    <div class="chart-box" :class="theme==='dark' ? 'dark' : 'light'" ref="chartBoxEl">
      <button v-if="isFullscreen" class="fs-exit" type="button" @click="toggleFullscreen">✖</button>
      <Bar v-if="displayMode==='combined'" :data="chartData" :options="chartOptions" />
      <div v-else class="split">
        <div class="pane price">
          <Line :data="priceChartData" :options="priceChartOptions" />
        </div>
        <div class="pane volume">
          <Bar :data="volumeChartData" :options="volumeChartOptions" />
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
  display: flex;
  gap: 16px;
  flex-wrap: wrap;
}
.file-label, .select-label {
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-weight: 600;
}
.chart-box {
  position: relative;
  width: 100%;
  height: 560px;
  background: #101014;
  border: 1px solid #2a2a2e;
  border-radius: 8px;
  padding: 8px;
}
.chart-box.light {
  background: #ffffff;
  border: 1px solid #dcdcdc;
}
.split {
  display: grid;
  grid-template-rows: 1fr 1fr;
  gap: 8px;
  height: 100%;
}
.pane { position: relative; }
input[type="file"], select {
  padding: 8px;
  background: #1a1a1e;
  border: 1px solid #2a2a2e;
  border-radius: 6px;
  color: #eaeaf0;
}
.fs-btn {
  padding: 8px 12px;
  background: #1a1a1e;
  border: 1px solid #2a2a2e;
  color: #eaeaf0;
  border-radius: 6px;
  cursor: pointer;
}
.theme-btn {
  padding: 8px 12px;
  background: #1a1a1e;
  border: 1px solid #2a2a2e;
  color: #eaeaf0;
  border-radius: 6px;
  cursor: pointer;
}
.fs-exit {
  position: absolute;
  top: 8px;
  right: 8px;
  background: rgba(0,0,0,0.5);
  color: #fff;
  border: 1px solid rgba(255,255,255,0.2);
  border-radius: 6px;
  padding: 4px 8px;
  cursor: pointer;
}

.color {
  width: 38px;
  height: 38px;
  padding: 0;
  background: transparent;
  border: 1px solid #2a2a2e;
  border-radius: 6px;
}

.mode-btn {
  padding: 8px 12px;
  background: #1a1a1e;
  border: 1px solid #2a2a2e;
  color: #eaeaf0;
  border-radius: 6px;
  cursor: pointer;
}
.mode-btn.active { background: #2a2a2e; }

</style>


