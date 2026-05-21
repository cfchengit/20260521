<template>
  <div class="view-layout">
    <aside class="sidebar">
      <div class="sidebar-header">
        <h2>🚲 YouBike 2.0 即時站點</h2>
        <div class="data-source">
          資料：<a href="https://tdx.transportdata.tw" target="_blank">TDX 交通部</a>
          ・地圖：<a href="https://www.openstreetmap.org" target="_blank">OpenStreetMap</a>
        </div>
      </div>

      <div class="city-wrap">
        <select class="city-select" v-model="selectedCity" @change="loadData">
          <option value="Taipei">台北市</option>
          <option value="NewTaipei">新北市</option>
          <option value="Taoyuan">桃園市</option>
          <option value="Taichung">台中市</option>
          <option value="Tainan">台南市</option>
          <option value="Kaohsiung">高雄市</option>
        </select>
        <button class="refresh-btn" @click="loadData" :disabled="isLoading">
          {{ isLoading ? '⏳' : '🔄' }}
        </button>
      </div>

      <div class="search-wrap">
        <input class="search-input" v-model="searchQuery"
          placeholder="🔍 搜尋站點名稱或地址..." />
      </div>

      <div class="filter-tabs">
        <button v-for="f in filters" :key="f.id" class="filter-btn"
          :class="{ active: currentFilter === f.id }"
          @click="currentFilter = f.id">{{ f.label }}</button>
      </div>

      <div class="status-msg" :class="statusType" v-if="statusMsg">{{ statusMsg }}</div>

      <div class="stats-row" v-if="mergedStations.length > 0">
        <div class="stat-card">
          <div class="stat-num blue">{{ filteredStations.length }}</div>
          <div class="stat-label">站點數</div>
        </div>
        <div class="stat-card">
          <div class="stat-num green">{{ totalBikes }}</div>
          <div class="stat-label">可借車輛</div>
        </div>
        <div class="stat-card">
          <div class="stat-num orange">{{ totalDocks }}</div>
          <div class="stat-label">可還空位</div>
        </div>
      </div>

      <div class="station-list">
        <div class="station-card"
          v-for="s in filteredStations" :key="s.uid"
          :class="{ selected: selectedUid === s.uid }"
          @click="focusStation(s)">
          <div class="station-top">
            <span class="station-name">{{ s.name }}</span>
            <span class="avail-badge" :class="availClass(s.bikes)">
              {{ availLabel(s.bikes) }}
            </span>
          </div>
          <div class="station-addr">📍 {{ s.addr }}</div>
          <div class="station-counts">
            <span class="count-bike">🚲 可借 <strong>{{ s.bikes }}</strong></span>
            <span class="count-dock">🅿️ 可還 <strong>{{ s.docks }}</strong></span>
            <span class="count-total">總 {{ s.total }}</span>
          </div>
          <div class="progress-bar">
            <div class="progress-fill"
              :style="{ width: bikeRatio(s) + '%' }"
              :class="progressClass(s)">
            </div>
          </div>
        </div>
        <div class="empty-state"
          v-if="!isLoading && mergedStations.length > 0 && filteredStations.length === 0">
          😢 找不到符合的站點
        </div>
        <div class="loading-state" v-if="isLoading">⏳ 載入中...</div>
      </div>

      <div class="update-bar" v-if="lastUpdate">🕐 更新：{{ lastUpdate }}</div>
    </aside>

    <div ref="mapEl" class="map-container"></div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import L from 'leaflet'

const mapEl     = ref(null)
const selectedCity    = ref('Taipei')
const mergedStations  = ref([])
const isLoading       = ref(false)
const lastUpdate      = ref('')
const searchQuery     = ref('')
const currentFilter   = ref('all')
const selectedUid     = ref(null)
const statusMsg       = ref('')
const statusType      = ref('')

let map = null
// ✅ 改用 Map 儲存 uid → marker，解決點擊找不到標記的問題
const markerMap = {}

let refreshTimer = null

const filters = [
  { id: 'all',      label: '全部' },
  { id: 'has_bike', label: '有車可借' },
  { id: 'has_dock', label: '有位可還' },
  { id: 'empty',    label: '目前無車' }
]

const filteredStations = computed(() => {
  let list = mergedStations.value
  if (searchQuery.value) {
    const q = searchQuery.value.toLowerCase()
    list = list.filter(s =>
      s.name?.toLowerCase().includes(q) ||
      s.addr?.toLowerCase().includes(q)
    )
  }
  if (currentFilter.value === 'has_bike') list = list.filter(s => s.bikes > 0)
  if (currentFilter.value === 'has_dock') list = list.filter(s => s.docks > 0)
  if (currentFilter.value === 'empty')    list = list.filter(s => s.bikes === 0)
  return list
})

const totalBikes = computed(() =>
  filteredStations.value.reduce((sum, s) => sum + s.bikes, 0)
)
const totalDocks = computed(() =>
  filteredStations.value.reduce((sum, s) => sum + s.docks, 0)
)

function initMap() {
  map = L.map(mapEl.value).setView([25.0478, 121.5319], 13)
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>',
    maxZoom: 19
  }).addTo(map)
}

async function loadData() {
  isLoading.value = true
  statusMsg.value = '📡 正在從 TDX 讀取 YouBike 資料...'
  statusType.value = 'loading'

  try {
    const city = selectedCity.value
    const base = 'https://tdx.transportdata.tw/api/basic/v2/Bike'

    const [stationRes, availRes] = await Promise.all([
      fetch(`${base}/Station/City/${city}?$top=2000&$format=JSON`),
      fetch(`${base}/Availability/City/${city}?$top=2000&$format=JSON`)
    ])

    if (!stationRes.ok || !availRes.ok) {
      throw new Error(`API 回應錯誤：Station=${stationRes.status}, Avail=${availRes.status}`)
    }

    const stationData = await stationRes.json()
    const availData   = await availRes.json()

    // ✅ 修正：TDX 兩支 API 的 StationUID 格式可能不同
    // Station API：  StationUID = "Taipei500101001"
    // Availability： StationUID = "TPE500101001" 或其他格式
    // 解法：同時建立「完整 UID」和「純數字 ID」兩個 Map
    const availByUID = {}
    const availByID  = {}
    availData.forEach(a => {
      // 完整 UID
      availByUID[a.StationUID] = a
      // 只取數字部分（去掉城市前綴）當備用 key
      const numId = a.StationUID?.replace(/[^0-9]/g, '')
      if (numId) availByID[numId] = a
    })

    const merged = stationData
      .filter(s => s.StationPosition?.PositionLat && s.StationPosition?.PositionLon)
      .map(s => {
        // 先用完整 UID 找，找不到就用純數字 ID 找
        const numId = s.StationUID?.replace(/[^0-9]/g, '')
        const avail = availByUID[s.StationUID] || availByID[numId] || {}

        // ✅ 修正：用 parseInt 確保數值型別正確，避免 NaN
        const bikes = parseInt(avail.AvailableRentBikes   ?? 0, 10)
        const docks = parseInt(avail.AvailableReturnBikes ?? 0, 10)
        const total = parseInt(s.BikesCapacity            ?? 0, 10)

        return {
          uid:    s.StationUID,
          name:   s.StationName?.Zh_tw   ?? '未知站點',
          addr:   s.StationAddress?.Zh_tw ?? '',
          lat:    parseFloat(s.StationPosition.PositionLat),
          lng:    parseFloat(s.StationPosition.PositionLon),
          total:  isNaN(total) ? 0 : total,
          bikes:  isNaN(bikes) ? 0 : bikes,
          docks:  isNaN(docks) ? 0 : docks,
          active: avail.ServiceAvailable ?? 1
        }
      })

    mergedStations.value = merged
    lastUpdate.value = new Date().toLocaleTimeString('zh-TW')

    const totalBikesNow = merged.reduce((s, st) => s + st.bikes, 0)
    statusMsg.value  = `✅ 共 ${merged.length} 站，目前可借 ${totalBikesNow} 輛`
    statusType.value = 'success'

    renderMarkers()

  } catch (err) {
    statusMsg.value  = `❌ 載入失敗：${err.message}`
    statusType.value = 'error'
    console.error('[YouBike] 載入失敗：', err)
  } finally {
    isLoading.value = false
  }
}

function renderMarkers() {
  // 清除舊標記
  Object.values(markerMap).forEach(m => map.removeLayer(m))
  Object.keys(markerMap).forEach(k => delete markerMap[k])

  const bounds = []

  mergedStations.value.forEach(s => {
    const { uid, lat, lng, bikes, docks, total, name, addr } = s
    if (!lat || !lng || isNaN(lat) || isNaN(lng)) return

    const ratio = total > 0 ? (bikes / total) * 100 : 0
    const color = bikes === 0 ? '#ef4444'
                : bikes <= 3  ? '#f97316'
                : bikes <= 7  ? '#eab308'
                :               '#22c55e'

    const icon = L.divIcon({
      className: '',
      html: `<div style="
        width:30px;height:30px;border-radius:50%;
        background:${color};border:2px solid white;
        display:flex;align-items:center;justify-content:center;
        font-size:12px;font-weight:700;color:white;
        box-shadow:0 2px 6px rgba(0,0,0,0.4);font-family:sans-serif;
      ">${bikes}</div>`,
      iconSize: [30, 30],
      iconAnchor: [15, 15]
    })

    const marker = L.marker([lat, lng], { icon })
      .addTo(map)
      .bindPopup(`
        <div style="font-family:'Noto Sans TC',sans-serif;min-width:200px;padding:4px">
          <strong>🚲 ${name}</strong><br>
          <small style="color:#888">📍 ${addr}</small><br><br>
          <div style="display:flex;gap:16px;font-size:0.9rem">
            <span style="color:#22c55e;font-weight:700">可借：${bikes} 輛</span>
            <span style="color:#3b82f6;font-weight:700">可還：${docks} 位</span>
          </div>
          <div style="background:#eee;border-radius:4px;height:6px;margin-top:8px;overflow:hidden">
            <div style="width:${ratio.toFixed(0)}%;height:100%;background:${color};border-radius:4px"></div>
          </div>
          <small style="color:#888">總車位：${total}</small>
        </div>
      `)

    marker.on('click', () => { selectedUid.value = uid })

    // ✅ 用 uid 當 key 存標記，點擊時可以直接用 uid 找到
    markerMap[uid] = marker
    bounds.push([lat, lng])
  })

  if (bounds.length > 0) {
    map.fitBounds(bounds, { padding: [20, 20] })
  }
}

// ✅ 修正：直接用 uid 從 markerMap 找標記，不再用陣列索引
function focusStation(s) {
  selectedUid.value = s.uid
  if (!s.lat || !s.lng) return
  map.setView([s.lat, s.lng], 17)
  const marker = markerMap[s.uid]
  if (marker) {
    marker.openPopup()
  }
}

function bikeRatio(s) {
  if (!s.total || s.total === 0) return 0
  return Math.round((s.bikes / s.total) * 100)
}
function progressClass(s) {
  const r = bikeRatio(s)
  if (r === 0)  return 'red'
  if (r <= 20)  return 'orange'
  if (r <= 50)  return 'yellow'
  return 'green'
}
function availClass(count) {
  if (count === 0) return 'badge-empty'
  if (count <= 3)  return 'badge-low'
  return 'badge-ok'
}
function availLabel(count) {
  if (count === 0) return '無車'
  if (count <= 3)  return '少量'
  return '充足'
}

onMounted(() => {
  initMap()
  loadData()
  refreshTimer = setInterval(loadData, 60000)
})
onUnmounted(() => {
  clearInterval(refreshTimer)
  map?.remove()
})
</script>

<style scoped>
.view-layout { display:flex; width:100%; height:100%; overflow:hidden; }

.sidebar {
  width:340px; flex-shrink:0; background:#1e293b;
  border-right:1px solid #334155; display:flex; flex-direction:column; overflow:hidden;
}
.sidebar-header { padding:14px 16px 10px; border-bottom:1px solid #334155; flex-shrink:0; }
.sidebar-header h2 { font-size:0.95rem; font-weight:700; color:#f1f5f9; }
.data-source { font-size:0.7rem; color:#64748b; margin-top:4px; }
.data-source a { color:#3b82f6; text-decoration:none; }

.city-wrap { display:flex; gap:8px; padding:10px 16px; flex-shrink:0; }
.city-select {
  flex:1; background:#0f172a; border:1px solid #334155; border-radius:8px;
  padding:8px 10px; color:#e2e8f0; font-size:0.85rem;
  font-family:'Noto Sans TC',sans-serif; cursor:pointer;
}
.city-select:focus { outline:none; border-color:#3b82f6; }
.refresh-btn {
  background:#0f172a; border:1px solid #334155; border-radius:8px;
  padding:8px 12px; color:#94a3b8; cursor:pointer; font-size:1rem; transition:all 0.2s;
}
.refresh-btn:hover:not(:disabled) { border-color:#3b82f6; color:#3b82f6; }
.refresh-btn:disabled { opacity:0.5; cursor:not-allowed; }

.search-wrap { padding:0 16px 8px; flex-shrink:0; }
.search-input {
  width:100%; background:#0f172a; border:1px solid #334155; border-radius:8px;
  padding:8px 12px; color:#e2e8f0; font-size:0.85rem;
  font-family:'Noto Sans TC',sans-serif;
}
.search-input:focus { outline:none; border-color:#3b82f6; }
.search-input::placeholder { color:#475569; }

.filter-tabs { display:flex; gap:5px; padding:0 16px 8px; flex-shrink:0; flex-wrap:wrap; }
.filter-btn {
  padding:4px 10px; border:1px solid #334155; border-radius:20px;
  background:transparent; color:#64748b; font-size:0.72rem;
  cursor:pointer; font-family:'Noto Sans TC',sans-serif; transition:all 0.2s;
}
.filter-btn.active { background:#3b82f6; border-color:#3b82f6; color:white; }

.status-msg {
  margin:0 16px 8px; border-radius:8px; padding:8px 12px; font-size:0.78rem; flex-shrink:0;
}
.status-msg.loading { background:rgba(59,130,246,.08); border:1px solid #3b82f6; color:#93c5fd; }
.status-msg.success { background:rgba(34,197,94,.08);  border:1px solid #22c55e; color:#86efac; }
.status-msg.error   { background:rgba(239,68,68,.08);  border:1px solid #ef4444; color:#f87171; }

.stats-row { display:grid; grid-template-columns:repeat(3,1fr); gap:6px; padding:0 16px 8px; flex-shrink:0; }
.stat-card { background:#0f172a; border:1px solid #334155; border-radius:8px; padding:8px; text-align:center; }
.stat-num { font-size:1.3rem; font-weight:700; line-height:1; }
.stat-num.blue { color:#3b82f6; } .stat-num.green { color:#22c55e; } .stat-num.orange { color:#f97316; }
.stat-label { font-size:0.65rem; color:#64748b; margin-top:4px; }

.station-list { flex:1; overflow-y:auto; padding:0 16px 8px; display:flex; flex-direction:column; gap:7px; }
.station-card {
  background:#0f172a; border:1px solid #334155; border-radius:10px;
  padding:10px 12px; cursor:pointer; transition:all 0.2s;
}
.station-card:hover { border-color:#3b82f6; }
.station-card.selected { border-color:#3b82f6; background:rgba(59,130,246,.08); }

.station-top { display:flex; justify-content:space-between; align-items:flex-start; margin-bottom:3px; gap:8px; }
.station-name { font-size:0.83rem; font-weight:600; color:#e2e8f0; flex:1; }
.avail-badge { font-size:0.65rem; padding:2px 7px; border-radius:20px; font-weight:600; flex-shrink:0; }
.badge-ok    { background:rgba(34,197,94,.15);  color:#86efac; }
.badge-low   { background:rgba(249,115,22,.15); color:#fdba74; }
.badge-empty { background:rgba(239,68,68,.15);  color:#fca5a5; }

.station-addr { font-size:0.7rem; color:#64748b; margin-bottom:6px; }
.station-counts { display:flex; gap:10px; margin-bottom:6px; align-items:center; }
.count-bike { font-size:0.78rem; color:#22c55e; }
.count-dock { font-size:0.78rem; color:#3b82f6; }
.count-total { font-size:0.7rem; color:#475569; margin-left:auto; }

.progress-bar { height:4px; background:#1e293b; border-radius:4px; overflow:hidden; }
.progress-fill { height:100%; border-radius:4px; transition:width 0.5s ease; }
.progress-fill.green  { background:#22c55e; }
.progress-fill.yellow { background:#eab308; }
.progress-fill.orange { background:#f97316; }
.progress-fill.red    { background:#ef4444; }

.empty-state, .loading-state { text-align:center; color:#64748b; padding:40px 0; font-size:0.9rem; }
.update-bar { padding:8px 16px; border-top:1px solid #334155; font-size:0.7rem; color:#475569; flex-shrink:0; }
.map-container { flex:1; }
</style>
