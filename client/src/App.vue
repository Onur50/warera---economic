<script setup>
import { ref } from 'vue'
import TradeHourlyBars from './components/TradeHourlyBars.vue'
import TaxPayersChart from './components/TaxPayersChart.vue'

const activeTab = ref('hourly')
</script>

<template>
  <div class="app-container">
    <div class="tabs">
      <button 
        class="tab-btn" 
        :class="{ active: activeTab === 'hourly' }"
        @click="activeTab = 'hourly'"
      >
        Hourly Trade Data
      </button>
      <button 
        class="tab-btn" 
        :class="{ active: activeTab === 'tax' }"
        @click="activeTab = 'tax'"
      >
        Tax Payers
      </button>
    </div>
    
    <div class="tab-content">
      <TradeHourlyBars v-if="activeTab === 'hourly'" />
      <TaxPayersChart v-if="activeTab === 'tax'" />
    </div>
  </div>
</template>

<style scoped>
.app-container {
  width: 100%;
  min-height: 100vh;
}
.tabs {
  position: fixed;
  top: 12px;
  right: 12px;
  z-index: 1001;
  display: flex;
  gap: 8px;
  background: rgba(26, 26, 30, 0.9);
  padding: 4px;
  border-radius: 8px;
  border: 1px solid #2a2a2e;
}
.tab-btn {
  padding: 6px 12px;
  background: transparent;
  border: none;
  border-radius: 6px;
  color: #a9abb3;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.2s;
}
.tab-btn:hover {
  background: rgba(255, 255, 255, 0.05);
  color: #eaeaf0;
}
.tab-btn.active {
  background: #42b883;
  color: #fff;
}
.tab-content {
  width: 100%;
}
</style>
