<template>
  <div class="view-layout">

    <!-- 左側面板 -->
    <aside class="sidebar">
      <div class="sidebar-header">
        <h2>🚌 公車即時位置</h2>
        <div class="data-source">
          資料來源：
          <a href="https://tdx.transportdata.tw" target="_blank">TDX 交通部運輸資料平台</a>
          （免費・無需申請）
        </div>
      </div>

      <div class="sidebar-body">

        <!-- 查詢表單 -->
        <div class="section-label">路線查詢</div>

        <div class="input-wrap">
          <label class="input-label">縣市</label>
          <select class="input-field" v-model="selectedCity">
            <option value="Taipei">台北市</option>
            <option value="NewTaipei">新北市</option>
            <option value="Taoyuan">桃園市</option>
            <option value="Taichung">台中市</option>
            <option value="Tainan">台南市</option>
            <option value="Kaohsiung">高雄市</option>
            <option value="Keelung">基隆市</option>
            <option value="Hsinchu">新竹市</option>
            <option value="HsinchuCounty">新竹縣</option>
            <option value="MiaoliCounty">苗栗縣</option>
            <option value="ChanghuaCounty">彰化縣</option>
            <option value="NantouCounty">南投縣</option>
            <option value="YunlinCounty">雲林縣</option>
            <option value="IlanCounty">宜蘭縣</option>
            <option value="HualienCounty">花蓮縣</option>
            <option value="TaitungCounty">台東縣</option>
          </select>
        </div>

        <div class="input-wrap">
          <label class="input-label">路線編號</label>
          <input
            class="input-field"
            v-model="routeName"
            placeholder="例：307、藍27、0東"
            @keyup.enter="doSearch"
          />
        </div>

        <button class="btn-search" @click="doSearch" :disabled="isLoading || !routeName.trim()">
          <span v-if="isLoading" class="spin">⏳</span>
          <span v-else>🔍</span>
          {{ isLoading ? '查詢中...' : '查詢公車位置' }}
        </button>

        <button
          class="btn-auto"
          :class="{ active: isAutoOn }"
          @click="toggleAuto"
          v-if="buses.length > 0"
        >
          {{ isAutoOn ? '🟢 自動更新中（30秒）' : '⏸️ 開啟自動更新' }}
        </button>

        <!-- 狀態訊息 -->
        <div class="status-box" :class="statusType" v-if="statusMsg">
          {{ statusMsg }}
        </div>

        <!-- 統計 -->
        <div class="stats-row" v-if="buses.length > 0">
          <div class="stat-card">
            <div class="stat-num blue">{{ buses.length }}</div>
            <div class="stat-label">行駛車輛</div>
          </div>
          <div class="stat-card">
            <div class="stat-num green">{{ runningCount }}</div>
            <div class="stat-label">執勤中</div>
          </div>
          <div class="stat-card">
            <div class="stat-num orange">{{ stopsGo.length }}</div>
            <div class="stat-label">站牌數</div>
          </div>
        </div>

        <div class="divider" v-if="buses.length > 0"></div>

        <!-- 分頁 -->
        <div class="inner-tabs" v-if="buses.length > 0">
          <button
            v-for="t in innerTabs"
            :key="t.id"
            class="inner-tab"
            :class="{ active: innerTab === t.id }"
            @click="innerTab = t.id"
          >{{ t.label }}</button>
        </div>

        <!-- 公車列表 -->
        <div class="item-list" v-if="innerTab === 'buses' && buses.length > 0">
          <div
            class="bus-card"
            v-for="(bus, idx) in buses"
            :key="bus.PlateNumb"
            :class="{ selected: selectedBus === bus.PlateNumb }"
            @click="focusBus(bus, idx)"
          >
            <div class="bus-top">
              <span class="plate">{{ bus.PlateNumb }}</span>
              <span class="duty-badge" :class="dutyClass(bus.DutyStatus)">
                {{ dutyLabel(bus.DutyStatus) }}
              </span>
            </div>
            <div class="bus-meta">
              <span>{{ bus.Direction === 0 ? '➡️ 去程' : '⬅️ 返程' }}</span>
              <span>🕐 {{ formatTime(bus.SrcRecTime) }}</span>
            </div>
          </div>
        </div>

        <!-- 站牌列表 -->
        <div class="item-list" v-if="innerTab === 'stops' && stopsGo.length > 0">
          <div
            class="stop-row"
            v-for="(stop, idx) in stopsGo"
            :key="stop.StopUID"
            @click="focusStop(stop, idx)"
          >
            <div class="stop-line-wrap">
              <div class="stop-dot" :class="{ terminal: idx === 0 || idx === stopsGo.length - 1 }"></div>
              <div class="stop-connector" v-if="idx < stopsGo.length - 1"></div>
            </div>
            <span class="stop-name">{{ stop.StopName?.Zh_tw }}</span>
            <span class="stop-seq">{{ stop.StopSequence }}</span>
          </div>
        </div>

      </div>

      <!-- 更新時間 -->
      <div class="update-bar" v-if="lastUpdate">
        <span>🕐 更新：{{ lastUpdate }}</span>
        <div class="live-dot" v-if="isAutoOn"></div>
      </div>
    </aside>

    <!-- 地圖 -->
    <div ref="mapEl" class="map-container"></div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import L from 'leaflet'

// ── 狀態 ──
const mapEl = ref(null)
const selectedCity = ref('Taipei')
const routeName = ref('')
const isLoading = ref(false)
const statusMsg = ref('請輸入路線編號開始查詢')
const statusType = ref('default')
const buses = ref([])
const stops = ref([])
const lastUpdate = ref('')
const isAutoOn = ref(false)
const selectedBus = ref(null)
const innerTab = ref('buses')

const innerTabs = [
  { id: 'buses', label: `🚌 公車列表` },
  { id: 'stops', label: `📍 站牌列表` }
]

let map = null
let busMarkers = []
let stopMarkers = []
let routeLine = null
let autoTimer = null

// ── 計算屬性 ──
const runningCount = computed(() => buses.value.filter(b => b.DutyStatus === 1).length)

const stopsGo = computed(() =>
  stops.value
    .filter(s => s.Direction === 0)
    .sort((a, b) => a.StopSequence - b.StopSequence)
)

// ── 初始化地圖 ──
function initMap() {
  map = L.map(mapEl.value).setView([25.0330, 121.5654], 13)
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>',
    maxZoom: 19
  }).addTo(map)
}

// ── 查詢（TDX 不帶 Token，完全免費使用）──
async function doSearch() {
  if (!routeName.value.trim()) return
  isLoading.value = true
  setStatus('loading', '🚌 正在抓取公車即時位置...')

  try {
    const city = selectedCity.value
    const route = encodeURIComponent(routeName.value.trim())
    const base = 'https://tdx.transportdata.tw/api/basic/v2/Bus'

    // TDX 不帶 Token 也能呼叫（政府開放資料，免費）
    const [busRes, stopRes] = await Promise.all([
      fetch(`${base}/RealTimeByFrequency/City/${city}/${route}?$top=500&$format=JSON`),
      fetch(`${base}/StopOfRoute/City/${city}/${route}?$top=500&$format=JSON`)
    ])

    const busData  = await busRes.json()
    const stopData = await stopRes.json()

    if (!busData?.length) {
      setStatus('error', `❌ 找不到「${routeName.value}」的公車，請確認路線和縣市`)
      return
    }

    buses.value = busData

    // 攤平巢狀站牌結構：Direction 在外層，Stops[] 在裡層
    stops.value = (stopData || []).flatMap(r =>
      (r.Stops || []).map(s => ({ ...s, Direction: r.Direction }))
    )

    lastUpdate.value = new Date().toLocaleTimeString('zh-TW')
    setStatus('success', `✅ 找到 ${busData.length} 輛公車、${stopsGo.value.length} 個站牌`)

    renderMap()
    innerTab.value = 'buses'

  } catch (err) {
    setStatus('error', `❌ 查詢失敗：${err.message}`)
  } finally {
    isLoading.value = false
  }
}

// ── 渲染地圖 ──
function renderMap() {
  busMarkers.forEach(m => map.removeLayer(m))
  stopMarkers.forEach(m => map.removeLayer(m))
  if (routeLine) map.removeLayer(routeLine)
  busMarkers = []; stopMarkers = []

  const bounds = []

  // 站牌
  stopsGo.value.forEach((stop, idx) => {
    const lat = stop.StopPosition?.PositionLat
    const lng = stop.StopPosition?.PositionLon
    if (!lat || !lng) return

    const isTerminal = idx === 0 || idx === stopsGo.value.length - 1
    const size = isTerminal ? 14 : 9

    const icon = L.divIcon({
      className: '',
      html: `<div style="
        width:${size}px;height:${size}px;border-radius:50%;
        background:${isTerminal ? '#3b82f6' : '#475569'};
        border:2px solid ${isTerminal ? 'white' : '#1e293b'};
        box-shadow:0 1px 4px rgba(0,0,0,0.5);
      "></div>`,
      iconSize: [size, size],
      iconAnchor: [size/2, size/2]
    })

    const marker = L.marker([lat, lng], { icon, zIndexOffset: 100 })
      .addTo(map)
      .bindPopup(`<strong>📍 ${stop.StopName?.Zh_tw}</strong><br><small>站序：第 ${stop.StopSequence} 站</small>`)

    stopMarkers.push(marker)
    bounds.push([lat, lng])
  })

  // 路線連線
  if (stopsGo.value.length > 1) {
    const latlngs = stopsGo.value
      .filter(s => s.StopPosition?.PositionLat)
      .map(s => [s.StopPosition.PositionLat, s.StopPosition.PositionLon])

    routeLine = L.polyline(latlngs, {
      color: '#3b82f6', weight: 3, opacity: 0.5, dashArray: '6,6'
    }).addTo(map)
  }

  // 公車標記
  buses.value.forEach((bus, idx) => {
    const lat = bus.BusPosition?.PositionLat
    const lng = bus.BusPosition?.PositionLon
    if (!lat || !lng) return

    const icon = L.divIcon({
      className: '',
      html: `<div style="
        width:34px;height:34px;border-radius:50%;
        background:#22c55e;border:2px solid white;
        display:flex;align-items:center;justify-content:center;
        font-size:17px;box-shadow:0 2px 8px rgba(0,0,0,0.5);cursor:pointer;
      ">🚌</div>`,
      iconSize: [34, 34],
      iconAnchor: [17, 17]
    })

    const marker = L.marker([lat, lng], { icon, zIndexOffset: 1000 })
      .addTo(map)
      .bindPopup(`
        <strong>🚌 ${bus.PlateNumb}</strong><br>
        <small>
          方向：${bus.Direction === 0 ? '去程 ➡️' : '返程 ⬅️'}<br>
          狀態：${dutyLabel(bus.DutyStatus)}<br>
          更新：${formatTime(bus.SrcRecTime)}
        </small>
      `)

    marker.on('click', () => { selectedBus.value = bus.PlateNumb })
    busMarkers.push(marker)
    bounds.push([lat, lng])
  })

  if (bounds.length > 0) map.fitBounds(bounds, { padding: [30, 30] })
}

// ── 點擊互動 ──
function focusBus(bus, idx) {
  selectedBus.value = bus.PlateNumb
  const lat = bus.BusPosition?.PositionLat
  const lng = bus.BusPosition?.PositionLon
  if (!lat || !lng) return
  map.setView([lat, lng], 16)
  busMarkers[idx]?.openPopup()
}

function focusStop(stop, idx) {
  const lat = stop.StopPosition?.PositionLat
  const lng = stop.StopPosition?.PositionLon
  if (!lat || !lng) return
  map.setView([lat, lng], 17)
  stopMarkers[idx]?.openPopup()
}

// ── 自動更新 ──
function toggleAuto() {
  isAutoOn.value = !isAutoOn.value
  if (isAutoOn.value) {
    autoTimer = setInterval(doSearch, 30000)
  } else {
    clearInterval(autoTimer)
  }
}

// ── 輔助函式 ──
function setStatus(type, msg) {
  statusType.value = type
  statusMsg.value = msg
}

function dutyLabel(s) {
  return { 0: '待機', 1: '執勤中', 2: '結束' }[s] ?? '未知'
}

function dutyClass(s) {
  return { 0: 'wait', 1: 'run', 2: 'end' }[s] ?? 'end'
}

function formatTime(iso) {
  if (!iso) return '—'
  try { return new Date(iso).toLocaleTimeString('zh-TW') } catch { return iso }
}

onMounted(() => {
  initMap()
})

onUnmounted(() => {
  clearInterval(autoTimer)
  map?.remove()
})
</script>

<style scoped>
.view-layout { display: flex; width: 100%; height: 100%; overflow: hidden; }

.sidebar {
  width: 320px; flex-shrink: 0;
  background: #1e293b; border-right: 1px solid #334155;
  display: flex; flex-direction: column; overflow: hidden;
}

.sidebar-header { padding: 14px 16px 10px; border-bottom: 1px solid #334155; flex-shrink: 0; }
.sidebar-header h2 { font-size: 0.95rem; font-weight: 700; color: #f1f5f9; }
.data-source { font-size: 0.7rem; color: #64748b; margin-top: 4px; }
.data-source a { color: #3b82f6; text-decoration: none; }

.sidebar-body { flex: 1; overflow-y: auto; padding: 14px 16px; display: flex; flex-direction: column; gap: 10px; }

.section-label { font-size: 0.68rem; font-weight: 700; color: #64748b; text-transform: uppercase; letter-spacing: 1.5px; }

.input-wrap { display: flex; flex-direction: column; gap: 4px; }
.input-label { font-size: 0.78rem; color: #94a3b8; }
.input-field {
  background: #0f172a; border: 1px solid #334155; border-radius: 8px;
  padding: 8px 12px; color: #e2e8f0; font-size: 0.85rem; font-family: 'Noto Sans TC', sans-serif;
}
.input-field:focus { outline: none; border-color: #3b82f6; }
.input-field::placeholder { color: #475569; }
select.input-field { appearance: none; cursor: pointer; }

.btn-search {
  background: linear-gradient(135deg, #3b82f6, #2563eb); color: white; border: none;
  border-radius: 8px; padding: 10px; font-size: 0.88rem; font-weight: 600;
  cursor: pointer; font-family: 'Noto Sans TC', sans-serif;
  display: flex; align-items: center; justify-content: center; gap: 6px;
  transition: all 0.2s;
}
.btn-search:hover:not(:disabled) { background: linear-gradient(135deg, #2563eb, #1d4ed8); }
.btn-search:disabled { background: #334155; color: #475569; cursor: not-allowed; }

.btn-auto {
  background: transparent; border: 1px solid #334155; border-radius: 8px;
  padding: 8px; color: #94a3b8; font-size: 0.8rem; cursor: pointer;
  font-family: 'Noto Sans TC', sans-serif; transition: all 0.2s;
}
.btn-auto.active { border-color: #22c55e; color: #86efac; background: rgba(34,197,94,0.05); }

.status-box { border-radius: 8px; padding: 10px 12px; font-size: 0.78rem; line-height: 1.5; }
.status-box.default { background: #0f172a; border: 1px solid #334155; color: #64748b; }
.status-box.loading { background: rgba(59,130,246,0.05); border: 1px solid #3b82f6; color: #93c5fd; }
.status-box.success { background: rgba(34,197,94,0.05); border: 1px solid #22c55e; color: #86efac; }
.status-box.error   { background: rgba(239,68,68,0.05); border: 1px solid #ef4444; color: #f87171; }

.stats-row { display: grid; grid-template-columns: repeat(3,1fr); gap: 6px; }
.stat-card { background: #0f172a; border: 1px solid #334155; border-radius: 8px; padding: 8px; text-align: center; }
.stat-num { font-size: 1.3rem; font-weight: 700; }
.stat-num.blue { color: #3b82f6; }
.stat-num.green { color: #22c55e; }
.stat-num.orange { color: #f97316; }
.stat-label { font-size: 0.65rem; color: #64748b; margin-top: 4px; }

.divider { height: 1px; background: #334155; }

.inner-tabs { display: flex; gap: 6px; }
.inner-tab {
  flex: 1; padding: 6px; background: #0f172a; border: 1px solid #334155;
  border-radius: 6px; color: #64748b; font-size: 0.75rem; cursor: pointer;
  font-family: 'Noto Sans TC', sans-serif; transition: all 0.2s;
}
.inner-tab.active { border-color: #3b82f6; color: #3b82f6; background: rgba(59,130,246,0.08); }

.item-list { display: flex; flex-direction: column; gap: 6px; }

.bus-card {
  background: #0f172a; border: 1px solid #334155; border-radius: 8px;
  padding: 10px; cursor: pointer; transition: all 0.2s;
}
.bus-card:hover { border-color: #3b82f6; }
.bus-card.selected { border-color: #3b82f6; background: rgba(59,130,246,0.08); }
.bus-top { display: flex; justify-content: space-between; margin-bottom: 4px; }
.plate { font-size: 0.75rem; font-weight: 700; color: #93c5fd; background: #1e293b; padding: 2px 8px; border-radius: 4px; }
.duty-badge { font-size: 0.68rem; padding: 2px 7px; border-radius: 20px; font-weight: 600; }
.duty-badge.run  { background: rgba(34,197,94,0.15); color: #86efac; }
.duty-badge.wait { background: rgba(234,179,8,0.15); color: #fde68a; }
.duty-badge.end  { background: rgba(100,116,139,0.15); color: #94a3b8; }
.bus-meta { display: flex; justify-content: space-between; font-size: 0.72rem; color: #64748b; }

.stop-row {
  display: flex; align-items: flex-start; gap: 8px;
  padding: 4px; cursor: pointer; border-radius: 6px; transition: all 0.15s;
}
.stop-row:hover { background: rgba(59,130,246,0.05); padding-left: 8px; }
.stop-line-wrap { display: flex; flex-direction: column; align-items: center; flex-shrink: 0; width: 12px; padding-top: 4px; }
.stop-dot { width: 9px; height: 9px; border-radius: 50%; background: #334155; border: 2px solid #475569; }
.stop-dot.terminal { background: #3b82f6; border-color: #3b82f6; }
.stop-connector { width: 2px; height: 16px; background: #334155; }
.stop-name { font-size: 0.78rem; color: #94a3b8; flex: 1; padding-top: 2px; }
.stop-seq { font-size: 0.68rem; color: #475569; padding-top: 2px; }

.update-bar {
  padding: 8px 16px; border-top: 1px solid #334155;
  display: flex; justify-content: space-between; align-items: center;
  font-size: 0.7rem; color: #475569; flex-shrink: 0;
}
@keyframes pulse { 0%,100%{opacity:1} 50%{opacity:0.3} }
.live-dot { width: 6px; height: 6px; border-radius: 50%; background: #22c55e; animation: pulse 1.5s infinite; }

.map-container { flex: 1; }
.spin { display: inline-block; animation: spin 0.8s linear infinite; }
@keyframes spin { to { transform: rotate(360deg); } }
</style>
