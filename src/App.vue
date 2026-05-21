<template>
  <div id="app">
    <!-- 頂部標題列 -->
    <header class="header">
      <div class="header-left">
        <span class="header-icon">🗺️</span>
        <div>
          <div class="header-title">政府開放資料地圖</div>
          <div class="header-sub">Taiwan Open Data × Leaflet × OpenStreetMap｜完全免費・無需申請</div>
        </div>
      </div>
      <div class="badges">
        <span class="badge green">✅ 免費</span>
        <span class="badge blue">🏛️ 政府資料</span>
        <span class="badge gray">🗺️ OpenStreetMap</span>
      </div>
    </header>

    <!-- 功能切換標籤 -->
    <nav class="nav-tabs">
      <button
        v-for="tab in tabs"
        :key="tab.id"
        class="nav-tab"
        :class="{ active: currentTab === tab.id }"
        @click="currentTab = tab.id"
      >
        {{ tab.icon }} {{ tab.name }}
      </button>
    </nav>

    <!-- 頁面內容 -->
    <main class="main-content">
      <YouBikeView v-if="currentTab === 'youbike'" />
      <BusView     v-if="currentTab === 'bus'" />
    </main>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import YouBikeView from './views/YouBikeView.vue'
import BusView from './views/BusView.vue'

const currentTab = ref('youbike')

const tabs = [
  { id: 'youbike', name: 'YouBike 即時站點', icon: '🚲' },
  { id: 'bus',     name: '公車即時位置',     icon: '🚌' }
]
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;500;700&display=swap');

* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  font-family: 'Noto Sans TC', sans-serif;
  background: #0f172a;
  color: #e2e8f0;
  min-height: 100vh;
}

#app { display: flex; flex-direction: column; height: 100vh; }

/* ── 頂部標題 ── */
.header {
  background: linear-gradient(135deg, #1e293b, #0f172a);
  border-bottom: 1px solid #334155;
  padding: 12px 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-shrink: 0;
}
.header-left { display: flex; align-items: center; gap: 12px; }
.header-icon { font-size: 1.8rem; }
.header-title { font-size: 1.1rem; font-weight: 700; color: #f1f5f9; }
.header-sub { font-size: 0.72rem; color: #64748b; margin-top: 2px; }
.badges { display: flex; gap: 6px; flex-wrap: wrap; }
.badge {
  font-size: 0.7rem; font-weight: 600;
  padding: 3px 10px; border-radius: 20px;
}
.badge.green { background: rgba(34,197,94,0.15); color: #86efac; border: 1px solid rgba(34,197,94,0.3); }
.badge.blue  { background: rgba(59,130,246,0.15); color: #93c5fd; border: 1px solid rgba(59,130,246,0.3); }
.badge.gray  { background: rgba(100,116,139,0.15); color: #94a3b8; border: 1px solid rgba(100,116,139,0.3); }

/* ── 導覽標籤 ── */
.nav-tabs {
  display: flex;
  background: #1e293b;
  border-bottom: 1px solid #334155;
  flex-shrink: 0;
}
.nav-tab {
  padding: 12px 24px;
  background: none;
  border: none;
  border-bottom: 2px solid transparent;
  color: #64748b;
  font-size: 0.88rem;
  font-weight: 500;
  cursor: pointer;
  font-family: 'Noto Sans TC', sans-serif;
  transition: all 0.2s;
}
.nav-tab:hover { color: #94a3b8; }
.nav-tab.active { color: #3b82f6; border-bottom-color: #3b82f6; }

/* ── 主內容 ── */
.main-content { flex: 1; overflow: hidden; display: flex; }
</style>
