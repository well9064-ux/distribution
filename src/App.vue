<script setup>
import { computed, nextTick, onBeforeUnmount, ref, watch } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'
import { Box, Home, KeyRound, LayoutDashboard, LogOut, MapPin, Package, Settings, Truck, UserRound } from '@lucide/vue'

const loggedIn = ref(false)
const collapsed = ref(false)
const activeMenu = ref('대시보드')
const showPassword = ref(false)
const loginId = ref('')
const password = ref('')
const mapTypes = [
  { label: '일반지도', value: 'Base' },
  { label: '백지도', value: 'white' },
  { label: '야간지도', value: 'midnight' },
  { label: '위성지도', value: 'Satellite' },
  { label: '하이브리드', value: 'Hybrid' }
]
const mapType = ref('Satellite')
const mapOptions = ref(['적치 여부'])
const displayItems = ref(['자재', '블록', '방해요소', '운송자원', '주요시설'])
const DEFAULT_MAP_ROTATION = 38
const DEFAULT_MAP_ZOOM = 1
const savedMapView = (() => {
  try { return JSON.parse(localStorage.getItem('hanwha-map-view') || '{}') }
  catch { return {} }
})()
const mapZoom = ref(Number(savedMapView.zoom) || DEFAULT_MAP_ZOOM)
const mapRotation = ref(Number.isFinite(Number(savedMapView.rotation)) ? Number(savedMapView.rotation) : DEFAULT_MAP_ROTATION)
const mapContainer = ref(null)
const vworldMapElement = ref(null)
const vworldReady = ref(false)
const vworldKey = import.meta.env.VITE_VWORLD_API_KEY || ''
let vworldMap
let vworldLayers = []
const showSplitMapDialog = ref(false)
const selectedSplitParcelId = ref('')
const splitMapButtonElement = ref(null)
const splitMapDialogElement = ref(null)
const splitMaps = ref([])
const selectableSplitParcels = computed(() => physicalParcels.value.filter(parcel => !splitMaps.value.some(map => map.parcelId === parcel.id)))
const splitMapElements = new Map()
const splitLeafletMaps = new Map()
const splitLeafletLayers = new Map()
const splitMapResizeObservers = new Map()
let splitMapSequence = 0
const searchCode = ref('')
const focusedCode = ref('')
const searchMessage = ref('')
const selectedObject = ref(null)

const objectDetails = {
  'MAT-12': { type: '자재', title: 'MAT-12', rows: [['자재코드','MAT-12'],['적치지번','A-12'],['적치장소','A야드 3열'],['자재내역','선박용 강재 PLATE'],['자재그룹','후판 자재'],['현재상태','적치중'],['재고량','124 TON']] },
  'MAT-08': { type: '자재', title: 'MAT-08', rows: [['자재코드','MAT-08'],['적치지번','C-08'],['적치장소','C야드 1열'],['자재내역','배관용 PIPE'],['자재그룹','의장 자재'],['현재상태','입고완료'],['재고량','86 EA']] },
  'F-03': { type: '운송자원 · 지게차', title: 'F-03', rows: [['중기코드','HE-FL-003'],['중기형태','지게차'],['장비번호','FL-03'],['운전자','김민수'],['신호수','박준호'],['이동카테고리','의장'],['이동구분','자재'],['작업현황','이동중'],['작업대상','MAT-12']] },
  'CR-02': { type: '운송자원 · 크레인', title: 'CR-02', rows: [['중기코드','HE-CR-002'],['중기형태','크레인'],['장비번호','CR-02'],['운전자','이현우'],['신호수','최성민'],['이동카테고리','선각Assy'],['이동구분','블록'],['작업현황','대기중'],['작업대상','BLK-A12-01']] },
  'TP-07': { type: '운송자원 · TP', title: 'TP-07', rows: [['중기코드','HE-TP-007'],['중기형태','TP'],['장비번호','TP-07'],['운전자','정우진'],['신호수','한지훈'],['이동카테고리','치구류'],['이동구분','치구'],['작업현황','이동중'],['작업대상','JIG-2407']] },
  'A-12': { type: '블록', title: 'A-12', rows: [['블록코드','BLK-A12-01'],['적치지번','A-12'],['규격','18.4 × 12.6 × 8.2 m'],['중량','286 TON'],['공사번호','HN-26041'],['호선번호','HNOE-3101'],['공정률','82%'],['공정단계','대조립'],['작업부서','선체생산1부']] },
  'B-04': { type: '블록', title: 'B-04', rows: [['블록코드','BLK-B04-02'],['적치지번','B-04'],['규격','15.8 × 10.2 × 6.4 m'],['중량','214 TON'],['공사번호','HN-26042'],['호선번호','HNOE-3102'],['공정률','64%'],['공정단계','중조립'],['작업부서','선체생산2부']] },
  'C-08': { type: '블록', title: 'C-08', rows: [['블록코드','BLK-C08-03'],['적치지번','C-08'],['규격','21.2 × 14.8 × 9.1 m'],['중량','342 TON'],['공사번호','HN-26043'],['호선번호','HNOE-3103'],['공정률','39%'],['공정단계','소조립'],['작업부서','블록조립부']] }
}

const menus = [
  { label: '대시보드', icon: LayoutDashboard },
  { label: '지번관리', icon: MapPin },
  { label: '자재관리', icon: Package },
  { label: '블록관리', icon: Box },
  { label: '운송수단관리', icon: Truck },
  { label: '시스템관리', icon: Settings },
]
const landSubmenus = ['물리지번 목록', '논리지번 목록', '지번 목록', '활용계획']
const landMenuOpen = ref(false)
const physicalMapElement = ref(null)
const physicalMapContainer = ref(null)
const selectedPhysicalId = ref('PHY-001')
const draggingPhysicalIndex = ref(null)
const drawMode = ref(false)
const physicalGridSelection = ref(null)
let physicalMap
let physicalVworldLayers = []
let physicalDraftLayer
let physicalGridLayer
let physicalGridDragStart
let physicalPolygonLayers = []
const PHYSICAL_GRID_ROWS = 30
const PHYSICAL_GRID_COLUMNS = 40
const PHYSICAL_GRID_CENTER = [34.9012707,127.593559]
const PHYSICAL_GRID_LAT_STEP = 10 / 111320
const PHYSICAL_GRID_LNG_STEP = 10 / (111320 * Math.cos(PHYSICAL_GRID_CENTER[0] * Math.PI / 180))
const PHYSICAL_GRID_NORTH = PHYSICAL_GRID_CENTER[0] + (PHYSICAL_GRID_ROWS / 2 * PHYSICAL_GRID_LAT_STEP)
const PHYSICAL_GRID_WEST = PHYSICAL_GRID_CENTER[1] - (PHYSICAL_GRID_COLUMNS / 2 * PHYSICAL_GRID_LNG_STEP)
const physicalParcels = ref([
  { id:'PHY-001', name:'A 야드 북측 구역', startParcel:'광양읍 황길리 1320-1', endParcel:'광양읍 황길리 1320-18', logicalParcel:'L-A-01', enabled:true, createdAt:'2026-06-18', createdBy:'김관리', updatedAt:'2026-07-20', updatedBy:'이담당', color:'#ED7100', points:[[34.9032,127.5905],[34.9035,127.5940],[34.9017,127.5944],[34.9015,127.5908]] },
  { id:'PHY-002', name:'B 야드 중앙 구역', startParcel:'광양읍 황길리 1321-2', endParcel:'광양읍 황길리 1321-15', logicalParcel:'L-B-01', enabled:true, createdAt:'2026-06-21', createdBy:'박담당', updatedAt:'2026-07-19', updatedBy:'박담당', color:'#2A6DFC', points:[[34.9014,127.5910],[34.9016,127.5946],[34.8998,127.5948],[34.8997,127.5912]] },
  { id:'PHY-003', name:'C 야드 남측 구역', startParcel:'광양읍 황길리 1322-1', endParcel:'광양읍 황길리 1322-11', logicalParcel:'L-C-02', enabled:false, createdAt:'2026-06-25', createdBy:'최관리', updatedAt:'2026-07-16', updatedBy:'김관리', color:'#40B22C', points:[[34.8995,127.5915],[34.8997,127.5949],[34.8981,127.5951],[34.8980,127.5918]] },
  { id:'PHY-004', name:'자재 입고 대기 구역', startParcel:'광양읍 황길리 1323-3', endParcel:'광양읍 황길리 1323-9', logicalParcel:'L-IN-01', enabled:true, createdAt:'2026-07-02', createdBy:'정담당', updatedAt:'2026-07-21', updatedBy:'정담당', color:'#F98E02', points:[[34.9028,127.5950],[34.9030,127.5971],[34.9017,127.5973],[34.9016,127.5951]] }
])
const selectedPhysicalParcel = computed(() => physicalParcels.value.find(item => item.id === selectedPhysicalId.value))
const physicalDraftParcel = computed(() => physicalParcels.value.find(item => item.draft))
const physicalGridAddresses = computed(() => {
  if (!physicalGridSelection.value) return { start:'', end:'' }
  const { minRow, maxRow, minColumn, maxColumn } = physicalGridSelection.value
  return { start:'(0,0)', end:`(${maxColumn - minColumn},${maxRow - minRow})` }
})

const now = computed(() => new Intl.DateTimeFormat('ko-KR', {
  year: 'numeric', month: '2-digit', day: '2-digit', weekday: 'short'
}).format(new Date()))

async function login() {
  loggedIn.value = true
  await nextTick()
  initializeVworldMap()
}
async function selectMenu(label) {
  if (label === '지번관리') return selectLandSubmenu('물리지번 목록')
  activeMenu.value = label
  landMenuOpen.value = false
}
async function selectLandSubmenu(label) {
  activeMenu.value = label
  landMenuOpen.value = true
  if (label === '물리지번 목록') {
    await nextTick()
    initializePhysicalMap()
  }
}
function initializePhysicalMap() {
  if (!physicalMapElement.value || physicalMap) return
  physicalMap = L.map(physicalMapElement.value, { zoomControl: true }).setView([34.9012707,127.593559], 16)
  setPhysicalVworldLayer()
  physicalMap.on('mousedown', startPhysicalGridSelection)
  physicalMap.on('mousemove', updatePhysicalGridSelection)
  physicalMap.on('mouseup', finishPhysicalGridSelection)
  renderPhysicalPolygons()
  if (drawMode.value) renderPhysicalGrid()
}
function setPhysicalVworldLayer() {
  if (!physicalMap || !vworldKey) return
  physicalVworldLayers.forEach(layer => physicalMap.removeLayer(layer))
  physicalVworldLayers = []
  const layers = mapType.value === 'Hybrid' ? ['Satellite','Hybrid'] : [mapType.value]
  layers.forEach(layerName => {
    const layer = L.tileLayer(vworldTileUrl(layerName), { minZoom:6, maxZoom:19, attribution:'&copy; VWorld 공간정보 오픈플랫폼' }).addTo(physicalMap)
    physicalVworldLayers.push(layer)
  })
}
async function togglePhysicalFullscreen() {
  if (!document.fullscreenElement) await physicalMapContainer.value?.requestFullscreen()
  else await document.exitFullscreen()
  setTimeout(() => physicalMap?.invalidateSize(), 100)
}
function renderPhysicalPolygons() {
  if (!physicalMap) return
  physicalPolygonLayers.forEach(layer => physicalMap.removeLayer(layer))
  const selectedParcel = physicalParcels.value.find(parcel => parcel.id === selectedPhysicalId.value)
  physicalPolygonLayers = selectedParcel?.points?.length ? [selectedParcel].map(parcel => {
    const layer = L.polygon(parcel.points, { color:parcel.color, weight:4, fillColor:parcel.color, fillOpacity:parcel.enabled ? .24 : .08, dashArray:parcel.enabled ? undefined : '5 5' }).addTo(physicalMap)
    layer.bindTooltip(`${parcel.id} · ${parcel.name}`, { permanent:true, direction:'center', className:'parcel-map-label' })
    layer.on('click', () => selectPhysicalParcel(parcel.id))
    return layer
  }) : []
}
function selectPhysicalParcel(id) {
  selectedPhysicalId.value = id
  renderPhysicalPolygons()
  const parcel = physicalParcels.value.find(item => item.id === id)
  if (parcel && physicalMap) physicalMap.fitBounds(parcel.points, { padding:[60,60], maxZoom:18 })
}
function dragPhysicalStart(index) { draggingPhysicalIndex.value = index }
function dropPhysicalAt(index) {
  const from = draggingPhysicalIndex.value
  if (from === null || from === index) return
  const reordered = [...physicalParcels.value]
  const [moved] = reordered.splice(from,1)
  reordered.splice(index,0,moved)
  physicalParcels.value = reordered
  draggingPhysicalIndex.value = null
}
function createPhysicalId() {
  const used = new Set(physicalParcels.value.map(parcel => parcel.id))
  let id
  do { id = `PHY-${Math.floor(100 + Math.random() * 900)}` } while (used.has(id))
  return id
}
function currentKoreanDate() {
  return new Intl.DateTimeFormat('sv-SE', { timeZone:'Asia/Seoul' }).format(new Date())
}
function physicalGridCell(latlng) {
  const column = Math.floor((latlng.lng - PHYSICAL_GRID_WEST) / PHYSICAL_GRID_LNG_STEP)
  const row = Math.floor((PHYSICAL_GRID_NORTH - latlng.lat) / PHYSICAL_GRID_LAT_STEP)
  return {
    row:Math.min(PHYSICAL_GRID_ROWS - 1, Math.max(0, row)),
    column:Math.min(PHYSICAL_GRID_COLUMNS - 1, Math.max(0, column)),
  }
}
function physicalGridCellBounds(row, column) {
  const north = PHYSICAL_GRID_NORTH - row * PHYSICAL_GRID_LAT_STEP
  const south = north - PHYSICAL_GRID_LAT_STEP
  const west = PHYSICAL_GRID_WEST + column * PHYSICAL_GRID_LNG_STEP
  const east = west + PHYSICAL_GRID_LNG_STEP
  return { north, south, west, east }
}
function renderPhysicalGrid() {
  if (!physicalMap) return
  if (physicalGridLayer) physicalMap.removeLayer(physicalGridLayer)
  const lines = []
  const south = PHYSICAL_GRID_NORTH - PHYSICAL_GRID_ROWS * PHYSICAL_GRID_LAT_STEP
  const east = PHYSICAL_GRID_WEST + PHYSICAL_GRID_COLUMNS * PHYSICAL_GRID_LNG_STEP
  for (let row = 0; row <= PHYSICAL_GRID_ROWS; row += 1) {
    const lat = PHYSICAL_GRID_NORTH - row * PHYSICAL_GRID_LAT_STEP
    lines.push(L.polyline([[lat,PHYSICAL_GRID_WEST],[lat,east]], { color:'#ffffff', weight:1, opacity:.72, interactive:false }))
  }
  for (let column = 0; column <= PHYSICAL_GRID_COLUMNS; column += 1) {
    const lng = PHYSICAL_GRID_WEST + column * PHYSICAL_GRID_LNG_STEP
    lines.push(L.polyline([[PHYSICAL_GRID_NORTH,lng],[south,lng]], { color:'#ffffff', weight:1, opacity:.72, interactive:false }))
  }
  physicalGridLayer = L.featureGroup(lines).addTo(physicalMap)
  physicalGridLayer.bringToFront()
  physicalMap.fitBounds([[south,PHYSICAL_GRID_WEST],[PHYSICAL_GRID_NORTH,east]], { padding:[20,20], maxZoom:18 })
}
function setPhysicalGridSelection(start, end) {
  const minRow = Math.min(start.row, end.row)
  const maxRow = Math.max(start.row, end.row)
  const minColumn = Math.min(start.column, end.column)
  const maxColumn = Math.max(start.column, end.column)
  const first = physicalGridCellBounds(minRow, minColumn)
  const last = physicalGridCellBounds(maxRow, maxColumn)
  const points = [[first.north,first.west],[first.north,last.east],[last.south,last.east],[last.south,first.west]]
  physicalGridSelection.value = { minRow, maxRow, minColumn, maxColumn, points }
  if (physicalDraftLayer) physicalMap.removeLayer(physicalDraftLayer)
  physicalDraftLayer = L.polygon(points, { color:'#ED7100', weight:3, fillColor:'#ED7100', fillOpacity:.28, interactive:false }).addTo(physicalMap)
  physicalDraftLayer.bindTooltip(`${maxColumn - minColumn + 1} × ${maxRow - minRow + 1}칸 · 10m 격자`, { permanent:true, direction:'center', className:'parcel-map-label' })
}
function startPhysicalGridSelection(event) {
  if (!drawMode.value) return
  physicalGridDragStart = physicalGridCell(event.latlng)
  setPhysicalGridSelection(physicalGridDragStart, physicalGridDragStart)
}
function updatePhysicalGridSelection(event) {
  if (!drawMode.value || !physicalGridDragStart) return
  setPhysicalGridSelection(physicalGridDragStart, physicalGridCell(event.latlng))
}
function finishPhysicalGridSelection(event) {
  if (!drawMode.value || !physicalGridDragStart) return
  setPhysicalGridSelection(physicalGridDragStart, physicalGridCell(event.latlng))
  physicalGridDragStart = null
}
function clearPhysicalDraftLayers() {
  if (physicalDraftLayer && physicalMap) physicalMap.removeLayer(physicalDraftLayer)
  if (physicalGridLayer && physicalMap) physicalMap.removeLayer(physicalGridLayer)
  physicalDraftLayer = null
  physicalGridLayer = null
  physicalGridDragStart = null
}
function startPhysicalDrawing() {
  if (drawMode.value) return
  const id = createPhysicalId()
  const today = currentKoreanDate()
  physicalParcels.value.push({ id, name:'새 물리지번', startParcel:'', endParcel:'', logicalParcel:'미연계', enabled:true, createdAt:today, createdBy:'관리자', updatedAt:today, updatedBy:'관리자', color:'#ED7100', points:[], draft:true })
  selectedPhysicalId.value = id
  drawMode.value = true
  physicalGridSelection.value = null
  physicalMap?.dragging.disable()
  renderPhysicalPolygons()
  renderPhysicalGrid()
}
function resetPhysicalDrawing() {
  if (!drawMode.value || !physicalDraftParcel.value) return
  physicalDraftParcel.value.name = '새 물리지번'
  physicalDraftParcel.value.enabled = true
  physicalGridSelection.value = null
  if (physicalDraftLayer && physicalMap) physicalMap.removeLayer(physicalDraftLayer)
  physicalDraftLayer = null
}
function cancelPhysicalDrawing() {
  const draftIndex = physicalParcels.value.findIndex(parcel => parcel.draft)
  if (draftIndex >= 0) physicalParcels.value.splice(draftIndex, 1)
  drawMode.value = false
  physicalGridSelection.value = null
  clearPhysicalDraftLayers()
  physicalMap?.dragging.enable()
  selectedPhysicalId.value = physicalParcels.value[0]?.id || ''
  renderPhysicalPolygons()
}
function completePhysicalDrawing() {
  const draft = physicalDraftParcel.value
  const selection = physicalGridSelection.value
  if (!draft || !selection || !draft.name.trim()) return
  draft.name = draft.name.trim()
  draft.startParcel = physicalGridAddresses.value.start
  draft.endParcel = physicalGridAddresses.value.end
  draft.points = [...selection.points]
  draft.updatedAt = currentKoreanDate()
  delete draft.draft
  drawMode.value = false
  clearPhysicalDraftLayers()
  physicalGridSelection.value = null
  physicalMap?.dragging.enable()
  renderPhysicalPolygons()
  selectPhysicalParcel(draft.id)
}
function toggleItem(item) {
  displayItems.value = displayItems.value.includes(item)
    ? displayItems.value.filter(v => v !== item)
    : [...displayItems.value, item]
}
function toggleOption(option) {
  mapOptions.value = mapOptions.value.includes(option)
    ? mapOptions.value.filter(v => v !== option)
    : [...mapOptions.value, option]
}
function resetMapSettings() {
  mapType.value = 'Satellite'
  mapOptions.value = ['적치 여부']
  displayItems.value = ['자재', '블록', '방해요소', '운송자원', '주요시설']
  resetMapView()
}
function zoomMap(delta) {
  mapZoom.value = Math.min(2, Math.max(.75, Number((mapZoom.value + delta).toFixed(2))))
}
function handleMapWheel(event) {
  zoomMap(event.deltaY < 0 ? .05 : -.05)
}
function applyZoomInput(event) {
  const value = Number(event.target.value)
  const percent = Number.isFinite(value) ? Math.min(200, Math.max(75, Math.round(value))) : 100
  mapZoom.value = percent / 100
  event.target.value = percent
}
function rotateMap(direction = 1) {
  let next = mapRotation.value + (15 * direction)
  if (next > 180) next -= 360
  if (next <= -180) next += 360
  mapRotation.value = next
}
function applyRotationInput() {
  const value = Number(mapRotation.value)
  mapRotation.value = Number.isFinite(value) ? Math.min(180, Math.max(-180, Math.round(value))) : DEFAULT_MAP_ROTATION
}
function resetMapView() { mapZoom.value = DEFAULT_MAP_ZOOM; mapRotation.value = DEFAULT_MAP_ROTATION }
async function toggleFullscreen() {
  if (!document.fullscreenElement) await mapContainer.value?.requestFullscreen()
  else await document.exitFullscreen()
}
function vworldTileUrl(layer) {
  const extension = layer === 'Satellite' ? 'jpeg' : 'png'
  return `https://api.vworld.kr/req/wmts/1.0.0/${vworldKey}/${layer}/{z}/{y}/{x}.${extension}`
}
function setVworldLayer() {
  if (!vworldMap || !vworldKey) return
  vworldLayers.forEach(layer => vworldMap.removeLayer(layer))
  vworldLayers = []
  const layers = mapType.value === 'Hybrid' ? ['Satellite', 'Hybrid'] : [mapType.value]
  layers.forEach(layerName => {
    const layer = L.tileLayer(vworldTileUrl(layerName), {
      minZoom: 6, maxZoom: 19,
      attribution: '&copy; VWorld 공간정보 오픈플랫폼'
    }).addTo(vworldMap)
    vworldLayers.push(layer)
  })
  vworldReady.value = true
}
function initializeVworldMap() {
  if (!vworldKey || !vworldMapElement.value || vworldMap) return
  vworldMap = L.map(vworldMapElement.value, { zoomControl: false, attributionControl: true }).setView([34.9012707, 127.593559], 16)
  setVworldLayer()
}
async function openSplitMapDialog() {
  selectedSplitParcelId.value = selectableSplitParcels.value[0]?.id || ''
  showSplitMapDialog.value = true
  await nextTick()
  splitMapDialogElement.value?.focus()
}
async function closeSplitMapDialog() {
  showSplitMapDialog.value = false
  await nextTick()
  splitMapButtonElement.value?.focus()
}
function addSplitMap() {
  if (!selectedSplitParcelId.value) return
  splitMaps.value.push({
    id: `split-map-${++splitMapSequence}`,
    parcelId: selectedSplitParcelId.value,
    mapType: 'Satellite',
    zoom: 1,
    rotation: 0,
    left: null,
    top: null,
    width: null,
    height: null,
  })
  closeSplitMapDialog()
}
function setSplitMapElement(id, element) {
  if (!element) {
    splitMapElements.delete(id)
    return
  }
  splitMapElements.set(id, element)
  const item = splitMaps.value.find(map => map.id === id)
  if (item) initializeSplitMap(item)
}
function initializeSplitMap(item) {
  const element = splitMapElements.get(item.id)
  const parcel = physicalParcels.value.find(parcelItem => parcelItem.id === item.parcelId)
  if (!element || !parcel || splitLeafletMaps.has(item.id)) return
  const map = L.map(element, { zoomControl:false, attributionControl:true })
  splitLeafletMaps.set(item.id, map)
  refreshSplitMapLayers(item)
  map.fitBounds(parcel.points, { padding:[28,28], maxZoom:18 })
  setTimeout(() => {
    map.invalidateSize()
    map.fitBounds(parcel.points, { padding:[28,28], maxZoom:18 })
  }, 100)
  const resizeObserver = new ResizeObserver(() => {
    map.invalidateSize({ pan:false })
    clampSplitMapWindow(item.id)
  })
  resizeObserver.observe(element)
  splitMapResizeObservers.set(item.id, resizeObserver)
}
function refreshSplitMapLayers(item) {
  const map = splitLeafletMaps.get(item.id)
  const parcel = physicalParcels.value.find(parcelItem => parcelItem.id === item.parcelId)
  if (!map || !parcel) return
  ;(splitLeafletLayers.get(item.id) || []).forEach(layer => map.removeLayer(layer))
  const layers = []
  const baseLayers = item.mapType === 'Hybrid' ? ['Satellite','Hybrid'] : [item.mapType]
  baseLayers.forEach(layerName => {
    layers.push(L.tileLayer(vworldTileUrl(layerName), { minZoom:6, maxZoom:19, attribution:'&copy; VWorld 공간정보 오픈플랫폼' }).addTo(map))
  })
  layers.push(L.polygon(parcel.points, { color:parcel.color, weight:4, fillColor:parcel.color, fillOpacity:parcel.enabled ? .24 : .08, dashArray:parcel.enabled ? undefined : '5 5' }).addTo(map))
  splitLeafletLayers.set(item.id, layers)
}
function setSplitMapType(item, type) {
  item.mapType = type
  refreshSplitMapLayers(item)
}
function zoomSplitMap(item, delta) {
  item.zoom = Math.min(2, Math.max(.75, Number((item.zoom + delta).toFixed(2))))
}
function rotateSplitMap(item, direction) {
  let next = item.rotation + (15 * direction)
  if (next > 180) next -= 360
  if (next <= -180) next += 360
  item.rotation = next
}
function resetSplitMap(item) {
  item.zoom = 1
  item.rotation = 0
  const parcel = physicalParcels.value.find(parcelItem => parcelItem.id === item.parcelId)
  if (parcel) splitLeafletMaps.get(item.id)?.fitBounds(parcel.points, { padding:[28,28], maxZoom:18 })
}
async function toggleSplitMapFullscreen(item) {
  const container = document.getElementById(`${item.id}-window`)
  if (!document.fullscreenElement) await container?.requestFullscreen()
  else await document.exitFullscreen()
  setTimeout(() => splitLeafletMaps.get(item.id)?.invalidateSize(), 100)
}
function resizeSplitMapWindow(item, delta) {
  const container = document.getElementById(`${item.id}-window`)
  const dock = container?.parentElement
  if (!container || !dock) return
  const bounds = container.getBoundingClientRect()
  const width = Math.min(dock.clientWidth, Math.max(280, Math.round(bounds.width + delta)))
  const height = Math.min(window.innerHeight - 120, Math.max(240, Math.round(bounds.height + delta * .7)))
  item.width = width
  item.height = height
}
function clampSplitMapWindow(id) {
  if (window.matchMedia('(max-width: 820px)').matches) return
  const container = document.getElementById(`${id}-window`)
  const dock = container?.parentElement
  const item = splitMaps.value.find(map => map.id === id)
  if (!container || !dock || !item || item.left === null) return
  const bounds = container.getBoundingClientRect()
  const dockBounds = dock.getBoundingClientRect()
  const left = Math.min(Math.max(0, bounds.left - dockBounds.left), Math.max(0, dock.clientWidth - bounds.width))
  const top = Math.min(Math.max(0, bounds.top - dockBounds.top), Math.max(0, dock.clientHeight - bounds.height))
  if (left !== item.left) item.left = left
  if (top !== item.top) item.top = top
}
function startSplitMapDrag(item, event) {
  if (event.button !== 0 || event.target.closest('button')) return
  const container = document.getElementById(`${item.id}-window`)
  const dock = container?.parentElement
  if (!container || !dock) return
  event.preventDefault()
  const bounds = container.getBoundingClientRect()
  const dockBounds = dock.getBoundingClientRect()
  const startLeft = bounds.left - dockBounds.left
  const startTop = bounds.top - dockBounds.top
  const startX = event.clientX
  const startY = event.clientY
  const move = moveEvent => {
    const left = Math.min(Math.max(0, startLeft + moveEvent.clientX - startX), Math.max(0, dock.clientWidth - bounds.width))
    const top = Math.min(Math.max(0, startTop + moveEvent.clientY - startY), Math.max(0, dock.clientHeight - bounds.height))
    container.style.left = `${left}px`
    container.style.top = `${top}px`
    container.style.right = 'auto'
    container.style.bottom = 'auto'
  }
  const stop = () => {
    const finalBounds = container.getBoundingClientRect()
    const finalDockBounds = dock.getBoundingClientRect()
    item.left = Math.max(0, Math.round(finalBounds.left - finalDockBounds.left))
    item.top = Math.max(0, Math.round(finalBounds.top - finalDockBounds.top))
    window.removeEventListener('pointermove', move)
    window.removeEventListener('pointerup', stop)
  }
  window.addEventListener('pointermove', move)
  window.addEventListener('pointerup', stop, { once:true })
}
function startSplitMapResize(item, event, corner = 'bottom-right') {
  const container = document.getElementById(`${item.id}-window`)
  const dock = container?.parentElement
  if (!container || !dock) return
  event.preventDefault()
  const bounds = container.getBoundingClientRect()
  const dockBounds = dock.getBoundingClientRect()
  const startLeft = bounds.left - dockBounds.left
  const startTop = bounds.top - dockBounds.top
  const startX = event.clientX
  const startY = event.clientY
  const resize = moveEvent => {
    const deltaX = moveEvent.clientX - startX
    const deltaY = moveEvent.clientY - startY
    const minWidth = dock.clientWidth < 280 ? dock.clientWidth : 280
    const maxHeight = Math.min(dock.clientHeight, window.innerHeight - 120)
    const width = Math.min(dock.clientWidth, Math.max(minWidth, Math.round(bounds.width + (corner === 'top-left' ? -deltaX : deltaX))))
    const height = Math.min(maxHeight, Math.max(240, Math.round(bounds.height + (corner === 'top-left' ? -deltaY : deltaY))))
    container.style.width = `${width}px`
    container.style.height = `${height}px`
    if (corner === 'top-left') {
      container.style.left = `${Math.max(0, startLeft + bounds.width - width)}px`
      container.style.top = `${Math.max(0, startTop + bounds.height - height)}px`
      container.style.right = 'auto'
      container.style.bottom = 'auto'
    }
  }
  const stop = () => {
    const finalBounds = container.getBoundingClientRect()
    const finalDockBounds = dock.getBoundingClientRect()
    item.width = Math.round(finalBounds.width)
    item.height = Math.round(finalBounds.height)
    if (corner === 'top-left' || item.left !== null) {
      item.left = Math.max(0, Math.round(finalBounds.left - finalDockBounds.left))
      item.top = Math.max(0, Math.round(finalBounds.top - finalDockBounds.top))
    }
    window.removeEventListener('pointermove', resize)
    window.removeEventListener('pointerup', stop)
  }
  window.addEventListener('pointermove', resize)
  window.addEventListener('pointerup', stop, { once:true })
}
function destroySplitMap(id) {
  splitMapResizeObservers.get(id)?.disconnect()
  splitMapResizeObservers.delete(id)
  splitLeafletMaps.get(id)?.remove()
  splitLeafletMaps.delete(id)
  splitLeafletLayers.delete(id)
  splitMapElements.delete(id)
}
function removeSplitMap(id) {
  destroySplitMap(id)
  splitMaps.value = splitMaps.value.filter(map => map.id !== id)
}
function destroyAllSplitMaps() {
  splitMaps.value.forEach(item => destroySplitMap(item.id))
}
function destroyVworldMap() {
  vworldMap?.remove()
  vworldMap = undefined
  vworldLayers = []
  vworldReady.value = false
  destroyAllSplitMaps()
}
function destroyPhysicalMap() {
  physicalMap?.remove()
  physicalMap = undefined
  physicalVworldLayers = []
  physicalPolygonLayers = []
  physicalDraftLayer = undefined
  physicalGridLayer = undefined
  physicalGridDragStart = undefined
}
function focusMapObject() {
  const code = searchCode.value.trim().toUpperCase().replace(/\s+/g, '')
  const objects = {
    'MAT-12': '자재', 'MAT12': '자재', 'MAT-08': '자재', 'MAT08': '자재',
    'OBS-01': '방해요소', 'OBS01': '방해요소', 'OBS-02': '방해요소', 'OBS02': '방해요소',
    'G1': '주요시설', 'WH': '주요시설'
  }
  const canonical = { MAT12: 'MAT-12', MAT08: 'MAT-08', OBS01: 'OBS-01', OBS02: 'OBS-02' }
  if (!objects[code]) {
    focusedCode.value = ''
    searchMessage.value = '일치하는 객체 코드가 없습니다.'
    return
  }
  const item = objects[code]
  if (!displayItems.value.includes(item)) displayItems.value = [...displayItems.value, item]
  focusedCode.value = canonical[code] || code
  searchCode.value = focusedCode.value
  searchMessage.value = `${focusedCode.value} 객체를 찾았습니다.`
  mapZoom.value = 1.35
  mapRotation.value = DEFAULT_MAP_ROTATION
}
function clearObjectFocus() {
  searchCode.value = ''
  focusedCode.value = ''
  searchMessage.value = ''
  mapZoom.value = 1
}
function showObjectDetails(code) {
  selectedObject.value = objectDetails[code]
  focusedCode.value = code
}
watch(mapType, () => { setVworldLayer(); setPhysicalVworldLayer() })
watch(activeMenu, async (menu, previousMenu) => {
  if (previousMenu === '대시보드' && menu !== '대시보드') destroyVworldMap()
  if (previousMenu === '물리지번 목록' && menu !== '물리지번 목록') destroyPhysicalMap()

  await nextTick()
  if (menu === '대시보드') initializeVworldMap()
  if (menu === '물리지번 목록') initializePhysicalMap()
})
watch([mapZoom, mapRotation], ([zoom, rotation]) => {
  localStorage.setItem('hanwha-map-view', JSON.stringify({ zoom, rotation }))
})
onBeforeUnmount(() => { destroyVworldMap(); destroyPhysicalMap() })
</script>

<template>
  <main v-if="!loggedIn" class="login-page">
    <section class="login-brand" aria-label="서비스 소개">
      <div class="brand-lockup brand-lockup--light">
        <span class="brand-mark"><i></i><i></i><i></i></span>
        <span>Hanwha</span>
      </div>
      <div class="brand-copy">
        <p class="eyebrow">SMART YARD MANAGEMENT</p>
        <h1>더 정확하고 효율적인<br />스마트 야드 운영</h1>
        <p>입고부터 배치, 운송까지 야드 물류 현황을<br />하나의 화면에서 확인하세요.</p>
      </div>
      <div class="brand-circles" aria-hidden="true"><i></i><i></i><i></i></div>
    </section>

    <section class="login-panel">
      <form class="login-form" @submit.prevent="login">
        <div class="mobile-brand">Hanwha</div>
        <p class="eyebrow orange">HANWHA OCEAN ECOTECH</p>
        <h2>스마트 야드 물류 시스템</h2>
        <p class="login-help">서비스 이용을 위해 로그인해주세요.</p>

        <label for="login-id">아이디</label>
        <div class="field-wrap">
          <UserRound class="field-icon" :size="17" :stroke-width="1.8" aria-hidden="true" />
          <input id="login-id" v-model="loginId" autocomplete="username" placeholder="아이디를 입력해주세요" />
        </div>

        <label for="login-password">비밀번호</label>
        <div class="field-wrap">
          <KeyRound class="field-icon" :size="17" :stroke-width="1.8" aria-hidden="true" />
          <input id="login-password" v-model="password" :type="showPassword ? 'text' : 'password'" autocomplete="current-password" placeholder="비밀번호를 입력해주세요" />
          <button type="button" class="icon-button" :aria-label="showPassword ? '비밀번호 숨기기' : '비밀번호 보기'" @click="showPassword = !showPassword">{{ showPassword ? '◉' : '◎' }}</button>
        </div>

        <div class="login-options">
          <label class="check-label"><input type="checkbox" /> <span>아이디 저장</span></label>
          <span>임시 계정으로 로그인할 수 있습니다.</span>
        </div>
        <button class="primary-button login-button" type="submit">로그인</button>
        <p class="copyright">© Hanwha Ocean EcoTech. All rights reserved.</p>
      </form>
    </section>
  </main>

  <div v-else class="app-shell" :class="{ collapsed }">
    <aside class="sidebar">
      <div class="sidebar-brand">
        <div class="project-logo" title="한화오션에코텍 물류Twin">
          <span class="project-symbol" aria-hidden="true"><i></i></span>
          <template v-if="!collapsed"><strong>한화오션에코텍</strong><span class="logo-divider"></span><b>물류Twin</b></template>
        </div>
        <button class="collapse-button" :aria-label="collapsed ? '메뉴 펼치기' : '메뉴 접기'" @click="collapsed = !collapsed">{{ collapsed ? '›' : '‹' }}</button>
      </div>
      <nav class="menu-list" aria-label="주 메뉴">
        <template v-for="menu in menus" :key="menu.label">
          <button :class="{ active: activeMenu === menu.label }" :title="collapsed ? menu.label : undefined" :aria-label="menu.label" @click="selectMenu(menu.label)">
            <component :is="menu.icon" class="menu-icon" aria-hidden="true" /><span v-if="!collapsed">{{ menu.label }}</span><span v-if="menu.label === '지번관리' && !collapsed" class="submenu-arrow">{{ landMenuOpen ? '⌃' : '⌄' }}</span>
          </button>
          <div v-if="menu.label === '지번관리' && landMenuOpen && !collapsed" class="submenu-list">
            <button v-for="submenu in landSubmenus" :key="submenu" :class="{ active: activeMenu === submenu }" @click.stop="selectLandSubmenu(submenu)">{{ submenu }}</button>
          </div>
        </template>
      </nav>
      <div class="sidebar-footer">
        <div class="avatar">관</div>
        <div v-if="!collapsed"><strong>관리자</strong><small>시스템 관리자</small></div>
        <button v-if="!collapsed" class="logout" @click="loggedIn = false" aria-label="로그아웃"><LogOut aria-hidden="true" /></button>
      </div>
    </aside>

    <main class="workspace">
      <header class="topbar">
        <div><Home class="home-icon" aria-hidden="true" /><span>›</span><strong>{{ activeMenu }}</strong></div>
        <div class="topbar-meta"><span>{{ now }}</span><button class="topbar-logout" aria-label="로그아웃" @click="loggedIn = false"><LogOut aria-hidden="true" /></button><span class="user-chip">관</span><strong>관리자</strong></div>
      </header>

      <div v-if="activeMenu === '대시보드'" class="dashboard">
        <h1 class="sr-only">대시보드</h1>
        <section class="stats-grid" aria-label="주요 통계">
          <article class="stat-card"><div class="stat-icon orange-bg">◔</div><div><span>야드 점유율</span><strong>68.4<small>%</small></strong><p><b class="up">▲ 2.3%</b> 전일 대비</p></div></article>
          <article class="stat-card"><div class="stat-icon sand-bg">⬡</div><div><span>블록개소</span><strong>142<small>개</small></strong><p><b class="up">▲ 8개</b> 전일 대비</p></div></article>
          <article class="stat-card"><div class="stat-icon gray-bg">⇥</div><div><span>입고예정 자재</span><strong>38<small>건</small></strong><p><b class="down">▼ 5건</b> 전일 대비</p></div></article>
          <article class="stat-card equipment-card"><div class="stat-icon dark-bg">▰</div><div class="equipment-main"><span>운송자원 장비</span><strong>27<small>대</small></strong><p>현재 운영 장비</p></div><dl><div><dt>지게차</dt><dd>12</dd></div><div><dt>크레인</dt><dd>8</dd></div><div><dt>TP</dt><dd>7</dd></div></dl></article>
        </section>

        <section class="map-card">
          <div class="map-toolbar">
            <div><h2>야드 현황</h2><span>최종 업데이트 14:32</span></div>
            <div class="segmented" aria-label="지도 유형">
              <button v-for="type in mapTypes" :key="type.value" :class="{ selected: mapType === type.value }" @click="mapType = type.value">{{ type.label }}</button>
            </div>
          </div>
          <div class="map-area">
            <div ref="mapContainer" class="map-stage" :class="{ 'has-vworld': vworldReady }" @wheel.prevent="handleMapWheel">
            <div ref="vworldMapElement" class="vworld-map" :style="{ transform: `scale(${mapZoom}) rotate(${mapRotation}deg)` }"></div>
            <div class="mock-map" :style="{ transform: `scale(${mapZoom}) rotate(${mapRotation}deg)`, '--counter-rotation': `${-mapRotation}deg` }">
              <div class="yard-operation-zone">
                <div v-if="mapOptions.includes('CAD 도면')" class="cad-overlay"><i></i><i></i><i></i><i></i></div>
                <template v-if="displayItems.includes('자재')"><button type="button" class="map-object material ma1" :class="{ focused: focusedCode === 'MAT-12' }" title="MAT-12 · 입고 자재" @click="showObjectDetails('MAT-12')">12</button><button type="button" class="map-object material ma2" :class="{ focused: focusedCode === 'MAT-08' }" title="MAT-08 · 입고 자재" @click="showObjectDetails('MAT-08')">8</button></template>
                <template v-if="displayItems.includes('방해요소')"><span class="map-object obstacle ob1" :class="{ focused: focusedCode === 'OBS-01' }" title="OBS-01 · 통행 방해요소">!</span><span class="map-object obstacle ob2" :class="{ focused: focusedCode === 'OBS-02' }" title="OBS-02 · 통행 방해요소">!</span></template>
                <template v-if="displayItems.includes('주요시설')"><span class="map-object facility fa1" :class="{ focused: focusedCode === 'G1' }" title="G1 · 제1 게이트">G1</span><span class="map-object facility fa2" :class="{ focused: focusedCode === 'WH' }" title="WH · 자재 창고">WH</span></template>
              </div>
            </div>
              <div v-if="!vworldKey" class="vworld-key-notice"><strong>VWorld 지도 준비됨</strong><span>API 인증키를 등록하면 실제 지도가 표시됩니다.</span></div>
              <form class="map-search" role="search" @submit.prevent="focusMapObject">
                <label for="object-code">객체 코드 검색</label>
                <div><input id="object-code" v-model="searchCode" placeholder="예: MAT-12, G1" autocomplete="off" @input="searchMessage = ''" /><button type="submit">검색</button><button v-if="focusedCode" type="button" class="clear-search" aria-label="검색 포커스 해제" @click="clearObjectFocus">×</button></div>
                <p v-if="searchMessage" :class="{ error: !focusedCode }">{{ searchMessage }}</p>
              </form>
              <div class="map-view-controls" aria-label="지도 보기 제어">
                <button title="확대" aria-label="지도 확대" :disabled="mapZoom >= 2" @click="zoomMap(.25)">＋</button>
                <label class="zoom-input"><input :value="Math.round(mapZoom * 100)" type="number" min="75" max="200" step="5" aria-label="지도 확대 비율" @change="applyZoomInput" /><span>%</span></label>
                <button title="축소" aria-label="지도 축소" :disabled="mapZoom <= .75" @click="zoomMap(-.25)">−</button>
                <button title="왼쪽으로 15도 회전" aria-label="지도 반시계 방향 15도 회전" @click="rotateMap(-1)">↺</button>
                <button title="오른쪽으로 15도 회전" aria-label="지도 시계 방향 15도 회전" @click="rotateMap(1)">↻</button>
                <button title="원위치" aria-label="지도 보기 원위치" @click="resetMapView">⌂</button>
                <button ref="splitMapButtonElement" title="화면분할" aria-label="화면분할 서브 지도 추가" @click="openSplitMapDialog">▦</button>
                <button title="전체화면" aria-label="지도 전체화면 전환" @click="toggleFullscreen">⛶</button>
              </div>
              <label class="rotation-indicator">회전 <input v-model.number="mapRotation" type="number" min="-180" max="180" step="1" aria-label="지도 회전 각도" @change="applyRotationInput" /><span>°</span></label>
              <div v-if="showSplitMapDialog" class="split-map-dialog-backdrop" role="presentation" @click.self="closeSplitMapDialog" @keydown.esc="closeSplitMapDialog">
                <section ref="splitMapDialogElement" class="split-map-dialog" role="dialog" aria-modal="true" aria-labelledby="split-map-dialog-title" tabindex="-1">
                  <header><div><span>화면분할</span><h2 id="split-map-dialog-title">추가할 서브 MAP 선택</h2></div><button type="button" aria-label="서브 지도 선택 닫기" @click="closeSplitMapDialog">×</button></header>
                  <div class="split-map-dialog-body">
                    <label for="split-parcel-select">물리지번 목록</label>
                    <select id="split-parcel-select" v-model="selectedSplitParcelId" :disabled="!selectableSplitParcels.length">
                      <option value="" disabled>물리지번을 선택하세요</option>
                      <option v-for="parcel in selectableSplitParcels" :key="parcel.id" :value="parcel.id">{{ parcel.name }} · {{ parcel.id }}</option>
                    </select>
                    <p v-if="!selectableSplitParcels.length">등록된 모든 물리지번이 서브 지도에 열려 있습니다.</p>
                  </div>
                  <footer><button type="button" class="gray-button" @click="closeSplitMapDialog">취소</button><button type="button" class="primary-small" :disabled="!selectedSplitParcelId" @click="addSplitMap">서브 MAP 추가</button></footer>
                </section>
              </div>
              <div v-if="splitMaps.length" class="split-map-windows" aria-label="서브 지도 목록">
                <section v-for="(subMap, subMapIndex) in splitMaps" :id="`${subMap.id}-window`" :key="subMap.id" class="split-map-window" :style="{ left: subMap.left === null ? 'auto' : `${subMap.left}px`, top: subMap.top === null ? 'auto' : `${subMap.top}px`, right: subMap.left === null ? `${16 + subMapIndex * 24}px` : 'auto', bottom: subMap.top === null ? `${16 + subMapIndex * 24}px` : 'auto', width: subMap.width ? `${subMap.width}px` : undefined, height: subMap.height ? `${subMap.height}px` : undefined }" :aria-label="`${physicalParcels.find(parcel => parcel.id === subMap.parcelId)?.name} 서브 지도`">
                  <header @pointerdown="startSplitMapDrag(subMap, $event)"><div><span>{{ subMap.parcelId }}</span><strong>{{ physicalParcels.find(parcel => parcel.id === subMap.parcelId)?.name }}</strong></div><div class="split-window-actions"><button type="button" aria-label="서브 지도 창 축소" @click="resizeSplitMapWindow(subMap, -60)">−</button><button type="button" aria-label="서브 지도 창 확대" @click="resizeSplitMapWindow(subMap, 60)">＋</button><button type="button" aria-label="서브 지도 닫기" @click="removeSplitMap(subMap.id)">×</button></div></header>
                  <span class="split-map-resize-handle split-map-resize-handle--top-left" aria-hidden="true" @pointerdown="startSplitMapResize(subMap, $event, 'top-left')"></span>
                  <div class="split-map-type segmented" aria-label="서브 지도 유형">
                    <button v-for="type in mapTypes" :key="type.value" :class="{ selected: subMap.mapType === type.value }" @click="setSplitMapType(subMap, type.value)">{{ type.label }}</button>
                  </div>
                  <div class="split-map-canvas" @wheel.stop.prevent="zoomSplitMap(subMap, $event.deltaY < 0 ? .05 : -.05)">
                    <div :ref="element => setSplitMapElement(subMap.id, element)" class="split-map-leaflet" :style="{ transform: `scale(${subMap.zoom}) rotate(${subMap.rotation}deg)` }"></div>
                    <div class="split-map-controls map-view-controls" aria-label="서브 지도 보기 제어">
                      <button aria-label="서브 지도 확대" :disabled="subMap.zoom >= 2" @click="zoomSplitMap(subMap, .25)">＋</button>
                      <span>{{ Math.round(subMap.zoom * 100) }}%</span>
                      <button aria-label="서브 지도 축소" :disabled="subMap.zoom <= .75" @click="zoomSplitMap(subMap, -.25)">−</button>
                      <button aria-label="서브 지도 반시계 방향 회전" @click="rotateSplitMap(subMap, -1)">↺</button>
                      <button aria-label="서브 지도 시계 방향 회전" @click="rotateSplitMap(subMap, 1)">↻</button>
                      <button aria-label="서브 지도 원위치" @click="resetSplitMap(subMap)">⌂</button>
                      <button aria-label="서브 지도 전체화면 전환" @click="toggleSplitMapFullscreen(subMap)">⛶</button>
                    </div>
                    <span class="split-map-rotation">회전 {{ subMap.rotation }}°</span>
                  </div>
                  <span class="split-map-resize-handle split-map-resize-handle--bottom-right" aria-hidden="true" @pointerdown="startSplitMapResize(subMap, $event)"></span>
                </section>
              </div>
              <aside v-if="selectedObject" class="object-detail" aria-live="polite">
                <header><div><span>{{ selectedObject.type }}</span><h3>{{ selectedObject.title }} 상세정보</h3></div><button type="button" aria-label="상세정보 닫기" @click="selectedObject = null">×</button></header>
                <dl><div v-for="row in selectedObject.rows" :key="row[0]"><dt>{{ row[0] }}</dt><dd :class="{ status: row[0] === '현재상태' }">{{ row[1] }}</dd></div></dl>
              </aside>
            </div>
            <aside class="map-options">
              <section><h3>지도 옵션</h3><button class="text-button" @click="resetMapSettings">초기화</button><label v-for="option in ['차도 지번','CAD 도면','지번 표시','적치 여부']" :key="option" class="switch-row"><span><i class="option-icon"></i>{{ option }}</span><input type="checkbox" :checked="mapOptions.includes(option)" @change="toggleOption(option)" /></label></section>
              <section><h3>표시 항목</h3><label v-for="item in ['자재','블록','방해요소','운송자원','주요시설']" :key="item" class="switch-row"><span><i :class="`legend legend-${item}`"></i>{{ item }}</span><input type="checkbox" :checked="displayItems.includes(item)" @change="toggleItem(item)" /></label></section>
            </aside>
          </div>
        </section>
      </div>

      <div v-else-if="activeMenu === '물리지번 목록'" class="physical-page">
        <header class="physical-page-header"><div><h1>물리지번 목록</h1><p>야드의 실제 공간 영역을 등록하고 표시 순서를 관리합니다.</p></div><div class="physical-actions"><button v-if="!drawMode" class="outline-button" @click="startPhysicalDrawing">＋ 물리지번 추가</button><template v-else><span class="draw-guide">10m × 10m 격자에서 영역을 드래그하세요.</span><button class="gray-button" @click="resetPhysicalDrawing">초기화</button><button class="gray-button" @click="cancelPhysicalDrawing">취소</button></template></div></header>
        <div class="physical-layout">
          <aside class="physical-list-panel">
            <div class="panel-heading"><div><h2>물리지번</h2><span>{{ physicalParcels.length }}개</span></div><small>끌어서 표시 순서를 변경합니다.</small></div>
            <div class="physical-list">
              <template v-for="(parcel,index) in physicalParcels" :key="parcel.id">
                <div v-if="parcel.draft" class="physical-list-draft selected" draggable="true" @dragstart="dragPhysicalStart(index)" @dragover.prevent @drop="dropPhysicalAt(index)">
                  <span class="drag-handle" title="순서 변경">⠿</span><span class="draft-folder" aria-hidden="true">▰</span><label><span class="sr-only">물리지번 명칭</span><input v-model="parcel.name" maxlength="40" placeholder="물리지번 명칭 입력" /></label><b>작성중</b>
                </div>
                <button v-else draggable="true" :class="{ selected: selectedPhysicalId === parcel.id, dragging: draggingPhysicalIndex === index }" :aria-pressed="selectedPhysicalId === parcel.id" @dragstart="dragPhysicalStart(index)" @dragover.prevent @drop="dropPhysicalAt(index)" @click="selectPhysicalParcel(parcel.id)">
                  <span class="drag-handle" title="순서 변경">⠿</span><i :style="{ background: parcel.color }"></i><span><strong>{{ parcel.name }}</strong><small>{{ parcel.id }} · {{ parcel.logicalParcel }}</small></span><b :class="{ off: !parcel.enabled }">{{ parcel.enabled ? '사용' : '미사용' }}</b>
                </button>
              </template>
            </div>
          </aside>
          <section ref="physicalMapContainer" class="physical-map-panel" @wheel.prevent="handleMapWheel">
            <div ref="physicalMapElement" class="physical-map" :style="{ transform: `scale(${mapZoom}) rotate(${mapRotation}deg)` }"></div>
            <div class="physical-map-types segmented" aria-label="지도 유형"><button v-for="type in mapTypes" :key="type.value" :class="{ selected: mapType === type.value }" @click="mapType = type.value">{{ type.label }}</button></div>
            <div class="physical-map-controls map-view-controls" aria-label="지도 보기 제어">
              <button title="확대" aria-label="지도 확대" :disabled="mapZoom >= 2" @click="zoomMap(.25)">＋</button>
              <label class="zoom-input"><input :value="Math.round(mapZoom * 100)" type="number" min="75" max="200" step="5" aria-label="지도 확대 비율" @change="applyZoomInput" /><span>%</span></label>
              <button title="축소" aria-label="지도 축소" :disabled="mapZoom <= .75" @click="zoomMap(-.25)">−</button>
              <button title="왼쪽으로 15도 회전" aria-label="지도 반시계 방향 15도 회전" @click="rotateMap(-1)">↺</button>
              <button title="오른쪽으로 15도 회전" aria-label="지도 시계 방향 15도 회전" @click="rotateMap(1)">↻</button>
              <button title="원위치" aria-label="지도 보기 원위치" @click="resetMapView">⌂</button>
              <button title="전체화면" aria-label="지도 전체화면 전환" @click="togglePhysicalFullscreen">⛶</button>
            </div>
            <label class="physical-rotation rotation-indicator">회전 <input v-model.number="mapRotation" type="number" min="-180" max="180" step="1" aria-label="지도 회전 각도" @change="applyRotationInput" /><span>°</span></label>
            <div v-if="drawMode" class="drawing-badge">10m × 10m 격자 선택 모드</div>
          </section>
          <aside v-if="drawMode && physicalDraftParcel" class="physical-detail-panel physical-create-panel">
            <div class="detail-heading"><span>신규 등록</span><h2>{{ physicalDraftParcel.name || '물리지번 명칭 미입력' }}</h2><small>격자 영역을 지정하면 좌표가 자동 계산됩니다.</small></div>
            <dl class="physical-create-fields">
              <div><dt>ID</dt><dd><input :value="physicalDraftParcel.id" readonly aria-label="자동 생성 ID" /></dd></div>
              <div><dt>시작 지번</dt><dd><input :value="physicalGridAddresses.start" readonly placeholder="영역 지정 필요" aria-label="시작 지번" /></dd></div>
              <div><dt>끝 지번</dt><dd><input :value="physicalGridAddresses.end" readonly placeholder="영역 지정 필요" aria-label="끝 지번" /></dd></div>
              <div><dt>사용여부</dt><dd class="physical-radio-group"><label><input v-model="physicalDraftParcel.enabled" type="radio" :value="true" /> 사용</label><label><input v-model="physicalDraftParcel.enabled" type="radio" :value="false" /> 미사용</label></dd></div>
              <div><dt>생성일자</dt><dd><input :value="physicalDraftParcel.createdAt" readonly aria-label="생성일자" /></dd></div>
              <div><dt>생성자</dt><dd><input :value="physicalDraftParcel.createdBy" readonly aria-label="생성자" /></dd></div>
            </dl>
            <div class="physical-selection-summary" aria-live="polite"><strong v-if="physicalGridSelection">{{ physicalGridSelection.maxColumn - physicalGridSelection.minColumn + 1 }} × {{ physicalGridSelection.maxRow - physicalGridSelection.minRow + 1 }}칸 선택</strong><span v-if="physicalGridSelection">실제 크기 {{ (physicalGridSelection.maxColumn - physicalGridSelection.minColumn + 1) * 10 }}m × {{ (physicalGridSelection.maxRow - physicalGridSelection.minRow + 1) * 10 }}m</span><span v-else>지도에서 영역을 드래그해주세요.</span></div>
            <div class="detail-footer"><button class="primary-small" :disabled="!physicalGridSelection || !physicalDraftParcel.name.trim()" @click="completePhysicalDrawing">완료</button></div>
          </aside>
          <aside v-else-if="selectedPhysicalParcel" class="physical-detail-panel">
            <div class="detail-heading"><span>상세정보</span><h2>{{ selectedPhysicalParcel.name }}</h2><small>{{ selectedPhysicalParcel.id }}</small></div>
            <dl>
              <div><dt>ID</dt><dd>{{ selectedPhysicalParcel.id }}</dd></div>
              <div><dt>시작 지번</dt><dd>{{ selectedPhysicalParcel.startParcel }}</dd></div>
              <div><dt>끝 지번</dt><dd>{{ selectedPhysicalParcel.endParcel }}</dd></div>
              <div><dt>연계 논리지번</dt><dd><span class="logical-chip">{{ selectedPhysicalParcel.logicalParcel }}</span></dd></div>
              <div><dt>사용여부</dt><dd><span class="use-chip" :class="{ off: !selectedPhysicalParcel.enabled }">{{ selectedPhysicalParcel.enabled ? '사용' : '미사용' }}</span></dd></div>
              <div><dt>생성일자</dt><dd>{{ selectedPhysicalParcel.createdAt }}</dd></div>
              <div><dt>생성자</dt><dd>{{ selectedPhysicalParcel.createdBy }}</dd></div>
              <div><dt>최근수정일</dt><dd>{{ selectedPhysicalParcel.updatedAt }}</dd></div>
              <div><dt>수정자</dt><dd>{{ selectedPhysicalParcel.updatedBy }}</dd></div>
            </dl>
            <div class="detail-footer"><button class="gray-button">삭제</button><button class="primary-small">수정</button></div>
          </aside>
        </div>
      </div>

      <div v-else class="empty-page"><span class="empty-icon">▦</span><h1>{{ activeMenu }}</h1><p>이 메뉴는 다음 단계에서 구성할 예정입니다.</p><button class="outline-button" @click="activeMenu = '대시보드'">대시보드로 이동</button></div>
    </main>
  </div>
</template>
