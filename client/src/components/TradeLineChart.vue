<script setup>
import { ref, computed } from 'vue'
import { Line } from 'vue-chartjs'
import {
  Chart as ChartJS,
  Title,
  Tooltip,
  Legend,
  LineElement,
  PointElement,
  LinearScale,
  TimeScale,
  Filler
} from 'chart.js'
import 'chartjs-adapter-date-fns'
import Papa from 'papaparse'

ChartJS.register(Title, Tooltip, Legend, LineElement, PointElement, LinearScale, TimeScale, Filler)

const rawRows = ref([])
const selectedItem = ref('')
const availableItems = ref([])
const chartBoxEl = ref(null)

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

const groupedByItem = computed(() => {
  const map = new Map()
  for (const row of rawRows.value) {
    if (!map.has(row.item)) map.set(row.item, [])
    map.get(row.item).push(row)
  }
  // kronolojik sıraya al
  for (const list of map.values()) list.sort((a, b) => new Date(a.dateIso) - new Date(b.dateIso))
  return map
})

const volumePerItem = computed(() => {
  const totals = new Map()
  for (const [item, list] of groupedByItem.value.entries()) {
    totals.set(item, list.reduce((s, r) => s + r.amount, 0))
  }
  return totals
})

function colorForIndex(i) {
  const palette = [
    '#42b883', '#646cff', '#ff7f50', '#ffd166', '#ef476f', '#06d6a0', '#118ab2', '#8338ec', '#3a86ff', '#fb5607'
  ]
  return palette[i % palette.length]
}

function scaleLog(val, min, max, outMin, outMax) {
  const a = Math.log(1 + Math.max(0, val))
  const lo = Math.log(1 + Math.max(0, min))
  const hi = Math.log(1 + Math.max(0, max))
  if (hi - lo === 0) return (outMin + outMax) / 2
  const t = (a - lo) / (hi - lo)
  return outMin + t * (outMax - outMin)
}

const chartData = computed(() => {
  const itemKeys = [selectedItem.value].filter((x) => groupedByItem.value.has(x))

  const totals = volumePerItem.value
  const vols = itemKeys.map((k) => totals.get(k) ?? 0)
  const minVol = Math.min(...(vols.length ? vols : [0]))
  const maxVol = Math.max(...(vols.length ? vols : [0]))

  const datasets = itemKeys.map((item, idx) => {
    const rows = groupedByItem.value.get(item) ?? []
    const data = rows.map((r) => ({ x: new Date(r.dateIso), y: r.amount, amount: r.amount, money: r.money }))

    const stroke = Math.round(scaleLog(totals.get(item) ?? 0, minVol, maxVol, 1.5, 6))

    return {
      label: item,
      data,
      parsing: false,
      borderColor: colorForIndex(idx),
      backgroundColor: colorForIndex(idx) + '55',
      borderWidth: stroke,
      tension: 0.2,
      pointRadius: (ctx) => {
        const amt = ctx.raw?.amount ?? 1
        return Math.round(scaleLog(amt, 1, 1000, 2, 6))
      },
      pointHoverRadius: 6,
      fill: false
    }
  })

  return { datasets }
})

const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: { position: 'top', labels: { color: '#ced4da' } },
    title: { display: true, text: 'Zaman - Hacim (Hacme göre ölçekli çizgi)', color: '#eaeaf0'} ,
    tooltip: {
      backgroundColor: 'rgba(20,20,24,0.95)',
      borderColor: '#2a2a2e',
      borderWidth: 1,
      titleColor: '#eaeaf0',
      bodyColor: '#eaeaf0',
      callbacks: {
        label: (ctx) => {
          const p = ctx.raw
          const dateStr = new Date(p.x).toLocaleString()
          const unit = p && p.amount ? (Number(p.money) / Number(p.amount)) : Number(p.money)
          const unitStr = Number.isFinite(unit) ? unit.toLocaleString('tr-TR', { maximumFractionDigits: 2 }) : '—'
          const totalStr = Number.isFinite(Number(p.money)) ? Number(p.money).toLocaleString('tr-TR', { maximumFractionDigits: 2 }) : '—'
          return `${ctx.dataset.label} | Tarih: ${dateStr} | Hacim: ${p.y} | Birim Fiyat: ${unitStr} (Toplam: ${totalStr})`
        }
      }
    }
  },
  scales: {
    x: { type: 'time', time: { tooltipFormat: 'yyyy-MM-dd HH:mm:ss' }, title: { display: true, text: 'Zaman', color: '#a9abb3' }, ticks: { color: '#a9abb3' }, grid: { color: 'rgba(255,255,255,0.06)' } },
    y: { type: 'linear', title: { display: true, text: 'Hacim', color: '#a9abb3' }, ticks: { color: '#a9abb3' }, grid: { color: 'rgba(255,255,255,0.06)' } }
  },
  interaction: { mode: 'nearest', intersect: false }
}

function toggleFullscreen() {
  const el = chartBoxEl.value
  if (!el) return
  if (!document.fullscreenElement) {
    el.requestFullscreen?.()
  } else {
    document.exitFullscreen?.()
  }
}
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
    </div>

    <div class="chart-box" ref="chartBoxEl">
      <Line :data="chartData" :options="chartOptions" />
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
  height: 520px;
  background: #101014;
  border: 1px solid #2a2a2e;
  border-radius: 8px;
  padding: 8px;
}
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
</style>


