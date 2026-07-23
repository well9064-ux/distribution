<script setup>
import { computed, nextTick, onBeforeUnmount, ref, watch } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'
import 'leaflet-rotate'
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
const MIN_MAP_ZOOM = .4
const MAX_MAP_ZOOM = 2
const BASE_LEAFLET_ZOOM = 16
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
let dashboardObjectLayer
let dashboardSiteLayer
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
  { label: '지번관리', icon: MapPin, children: [
    { label: '물리지번 목록' },
    { label: '논리지번 목록' },
    { label: '지번목록 조회' },
    { label: '활용계획 관리' },
  ] },
  { label: '자재관리', icon: Package, children: [
    { label: '입고예정정보', children: [
      { label: '의장재' },
      { label: '기기장비재' },
      { label: '강재' },
    ] },
    { label: '재고가용성정보' },
    { label: '핵심자재관리' },
    { label: '적치위치관리' },
  ] },
  { label: '블록관리', icon: Box },
  { label: '운송수단관리', icon: Truck, children: [
    { label: '중기배차신청' },
    { label: '중기배차계획' },
    { label: '배차실적' },
    { label: '운송부하율관리', children: [
      { label: '부하율마스터관리' },
      { label: '운송자원별 부하율 분석/통계' },
    ] },
    { label: '장비가동률' },
  ] },
  { label: '시스템관리', icon: Settings, children: [
    { label: '히스토리 관리' },
    { label: '사용자권한 관리' },
  ] },
]
const openMenus = ref(['지번관리'])
const physicalMapElement = ref(null)
const physicalMapContainer = ref(null)
const selectedPhysicalId = ref('PHY-316')
const draggingPhysicalIndex = ref(null)
const drawMode = ref(false)
const editingPhysicalId = ref('')
const deleteConfirmPhysicalId = ref('')
const physicalGridSelection = ref(null)
const physicalGridAwaitingEnd = ref(false)
let physicalEditSnapshot
let physicalMap
let physicalVworldLayers = []
let physicalDraftLayer
let physicalGridLayer
let physicalGridDragStart
let physicalPolygonLayers = []
const PHYSICAL_GRID_CENTER = [34.9012707,127.593559]
const PHYSICAL_GRID_CELL_METERS = 10
const METERS_PER_LATITUDE_DEGREE = 111320
const METERS_PER_LONGITUDE_DEGREE = 111320 * Math.cos(PHYSICAL_GRID_CENTER[0] * Math.PI / 180)
const PHYSICAL_GRID_COLOR = '#FFE600'
const PHYSICAL_PARCELS_STORAGE_KEY = 'hanwha-physical-parcels-v1'
function sampleGridPoints(centerHorizontal,centerVertical,columns,rows) {
  const halfWidth = columns * PHYSICAL_GRID_CELL_METERS / 2
  const halfHeight = rows * PHYSICAL_GRID_CELL_METERS / 2
  const angle = DEFAULT_MAP_ROTATION * Math.PI / 180
  return [[-halfWidth,-halfHeight],[halfWidth,-halfHeight],[halfWidth,halfHeight],[-halfWidth,halfHeight]].map(([horizontal,vertical]) => {
    const localHorizontal = centerHorizontal + horizontal
    const localVertical = centerVertical + vertical
    const eastMeters = Math.cos(angle) * localHorizontal + Math.sin(angle) * localVertical
    const southMeters = -Math.sin(angle) * localHorizontal + Math.cos(angle) * localVertical
    return [PHYSICAL_GRID_CENTER[0] - southMeters / METERS_PER_LATITUDE_DEGREE,PHYSICAL_GRID_CENTER[1] + eastMeters / METERS_PER_LONGITUDE_DEGREE]
  })
}
const defaultPhysicalParcels = [
  { id:'PHY-316',name:'한화오션에코텍',startParcel:'(0,0)',endParcel:'(152,65)',logicalParcel:'미연계',enabled:true,createdAt:'2026-07-22',createdBy:'관리자',updatedAt:'2026-07-22',updatedBy:'관리자',gridColumns:153,gridRows:66,points:sampleGridPoints(0,0,153,66) },
  { id:'PHY-747',name:'임대지역',startParcel:'(0,0)',endParcel:'(27,42)',logicalParcel:'미연계',enabled:true,createdAt:'2026-07-22',createdBy:'관리자',updatedAt:'2026-07-22',updatedBy:'관리자',gridColumns:28,gridRows:43,points:sampleGridPoints(650,-300,28,43) },
]
function loadPhysicalParcels() {
  try {
    const stored = JSON.parse(localStorage.getItem(PHYSICAL_PARCELS_STORAGE_KEY) || 'null')
    return Array.isArray(stored) && stored.length ? stored : JSON.parse(JSON.stringify(defaultPhysicalParcels))
  } catch { return JSON.parse(JSON.stringify(defaultPhysicalParcels)) }
}
function persistPhysicalParcels() {
  localStorage.setItem(PHYSICAL_PARCELS_STORAGE_KEY,JSON.stringify(physicalParcels.value.filter(parcel => !parcel.draft)))
}
const physicalParcels = ref(loadPhysicalParcels())
if (!physicalParcels.value.some(parcel => parcel.id === selectedPhysicalId.value)) selectedPhysicalId.value = physicalParcels.value[0]?.id || ''
if (!localStorage.getItem(PHYSICAL_PARCELS_STORAGE_KEY)) persistPhysicalParcels()
const selectedPhysicalParcel = computed(() => physicalParcels.value.find(item => item.id === selectedPhysicalId.value))
const physicalDraftParcel = computed(() => physicalParcels.value.find(item => item.draft))
const dashboardObjects = [
  { code:'MAT-12', type:'자재', label:'12', u:.20, v:.70 },
  { code:'MAT-08', type:'자재', label:'8', u:.62, v:.24 },
  { code:'A-12', type:'블록', label:'A12', u:.30, v:.32 },
  { code:'B-04', type:'블록', label:'B04', u:.48, v:.68 },
  { code:'C-08', type:'블록', label:'C08', u:.72, v:.38 },
  { code:'OBS-01', type:'방해요소', label:'!', u:.39, v:.48 },
  { code:'OBS-02', type:'방해요소', label:'!', u:.72, v:.72 },
  { code:'F-03', type:'운송자원', label:'F3', u:.43, v:.42 },
  { code:'CR-02', type:'운송자원', label:'CR2', u:.58, v:.74 },
  { code:'TP-07', type:'운송자원', label:'TP7', u:.76, v:.54 },
  { code:'G1', type:'주요시설', label:'G1', u:.12, v:.45 },
  { code:'WH', type:'주요시설', label:'WH', u:.80, v:.36 },
]
const dashboardSiteParcel = computed(() => physicalParcels.value.find(parcel => !parcel.draft && parcel.enabled && parcel.name.trim() === '한화오션에코텍'))
const physicalGridAddresses = computed(() => {
  if (!physicalGridSelection.value) return { start:'', end:'' }
  if (editingPhysicalId.value && physicalDraftParcel.value) {
    return { start:physicalDraftParcel.value.startParcel,end:physicalDraftParcel.value.endParcel }
  }
  const { minRow, maxRow, minColumn, maxColumn } = physicalGridSelection.value
  return { start:'(0,0)', end:`(${maxColumn - minColumn},${maxRow - minRow})` }
})
const loadPlanView = ref('month')
const loadPlanPcg = ref('전체')
const loadPlanKeyword = ref('')
const loadPlanYear = ref(2026)
const tpLoadResourceSeeds = [
  { pcg:'HC', spec:'006', code:'CH006', standardHours:.6 },
  { pcg:'HC', spec:'010', code:'CH010', standardHours:.6 },
  { pcg:'HC', spec:'032', code:'CH032', standardHours:.6 },
  { pcg:'LNGC DK', spec:'021', code:'LD021', standardHours:.75 },
  { pcg:'LNGC DK', spec:'032', code:'LD032', standardHours:.75 },
  { pcg:'LNGC DK', spec:'050', code:'LD050', standardHours:1.4 },
  { pcg:'LNGC DK', spec:'240', code:'LD240', standardHours:3 },
  { pcg:'CMR', spec:'032', code:'LC032', standardHours:.75 },
  { pcg:'CMR', spec:'050', code:'LC050', standardHours:.7 },
  { pcg:'LPGC CMR', spec:'021', code:'PC021', standardHours:.75 },
  { pcg:'E/Room', spec:'021', code:'LE021', standardHours:.75 },
  { pcg:'E/Room', spec:'032', code:'LE032', standardHours:.75 },
  { pcg:'LNGC TBHD', spec:'032', code:'LT032', standardHours:.75 },
  { pcg:'LNGC TBHD', spec:'050', code:'LT050', standardHours:.7 },
  { pcg:'LNGC TBHD', spec:'240', code:'LT240', standardHours:2.5 },
  { pcg:'FD#6', spec:'050', code:'FD050', standardHours:.7 },
  { pcg:'FD#6', spec:'240', code:'FD240', standardHours:2 },
]
const LOAD_MASTER_STORAGE_KEY = 'hanwha-load-masters-v1'
const DISPATCH_PLAN_STORAGE_KEY = 'hanwha-dispatch-plans-v1'
const defaultEquipmentMasters = [
  { id:'EQ-021', code:'TP021', name:'21톤 TP', category:'TP', spec:'021', capacity:21, ownedCount:3, dailyHours:8, availability:90, enabled:true },
  { id:'EQ-032', code:'TP032', name:'32톤 TP', category:'TP', spec:'032', capacity:32, ownedCount:3, dailyHours:8, availability:90, enabled:true },
  { id:'EQ-050', code:'TP050', name:'50톤 TP', category:'TP', spec:'050', capacity:50, ownedCount:2, dailyHours:8, availability:85, enabled:true },
  { id:'EQ-240', code:'TP240', name:'240톤 TP', category:'TP', spec:'240', capacity:240, ownedCount:1, dailyHours:8, availability:85, enabled:true },
]
const defaultStandardTimeMasters = tpLoadResourceSeeds.map((item, index) => ({
  id:`ST-${String(index + 1).padStart(3,'0')}`, pcg:item.pcg, equipmentCode:`TP${item.spec}`, hours:item.standardHours,
}))
const defaultDispatchRuleMasters = [
  { id:'RL-001', pcg:'LNGC DK', stage:'중조', task:'중조주판 반출', triggerStage:'중조', triggerPoint:'착수', offsetDays:3, equipmentCode:'TP021', quantity:1 },
  { id:'RL-002', pcg:'LNGC DK', stage:'대조', task:'대조주판 배치', triggerStage:'대조', triggerPoint:'착수', offsetDays:1, equipmentCode:'TP050', quantity:1 },
  { id:'RL-003', pcg:'LNGC DK', stage:'대조', task:'중조 이동 탑재', triggerStage:'대조', triggerPoint:'착수', offsetDays:2, equipmentCode:'TP032', quantity:1 },
  { id:'RL-004', pcg:'CMR', stage:'도장', task:'전처리 입고', triggerStage:'도장', triggerPoint:'착수', offsetDays:-1, equipmentCode:'TP050', quantity:1 },
  { id:'RL-005', pcg:'LNGC TBHD', stage:'PE', task:'후PE 착수', triggerStage:'PE', triggerPoint:'착수', offsetDays:0, equipmentCode:'TP240', quantity:1 },
]
function loadStoredData(key, fallback) {
  try {
    const stored = JSON.parse(localStorage.getItem(key) || 'null')
    return stored || fallback
  } catch { return fallback }
}
const storedLoadMasters = loadStoredData(LOAD_MASTER_STORAGE_KEY, null)
const equipmentMasters = ref(storedLoadMasters?.equipment || JSON.parse(JSON.stringify(defaultEquipmentMasters)))
const standardTimeMasters = ref(storedLoadMasters?.standardTimes || JSON.parse(JSON.stringify(defaultStandardTimeMasters)))
const dispatchRuleMasters = ref(storedLoadMasters?.rules || JSON.parse(JSON.stringify(defaultDispatchRuleMasters)))
const defaultDispatchPlans = [
  { id:'DP-001', project:'HNOE-3101', block:'A12-01', ruleId:'RL-001', baseDate:'2026-07-06', status:'계획' },
  { id:'DP-002', project:'HNOE-3101', block:'A12-02', ruleId:'RL-002', baseDate:'2026-07-10', status:'확정' },
  { id:'DP-003', project:'HNOE-3102', block:'B04-01', ruleId:'RL-003', baseDate:'2026-08-03', status:'계획' },
  { id:'DP-004', project:'HNOE-3102', block:'B04-03', ruleId:'RL-004', baseDate:'2026-08-17', status:'확정' },
  { id:'DP-005', project:'HNOE-3103', block:'C08-02', ruleId:'RL-005', baseDate:'2026-09-07', status:'계획' },
]
const dispatchPlans = ref(loadStoredData(DISPATCH_PLAN_STORAGE_KEY, JSON.parse(JSON.stringify(defaultDispatchPlans))))
const masterTab = ref('rule')
const masterEditorOpen = ref(false)
const masterEditingId = ref('')
const masterForm = ref({})
const masterNotice = ref('')
const dispatchEditorOpen = ref(false)
const dispatchEditingId = ref('')
const dispatchForm = ref({})
const dispatchNotice = ref('')
const dispatchStageFilter = ref('전체')
const loadResources = computed(() => standardTimeMasters.value.map(item => {
  const equipment = equipmentMasters.value.find(master => master.code === item.equipmentCode)
  return { pcg:item.pcg, spec:equipment?.spec || item.equipmentCode.replace(/\D/g,''), code:item.equipmentCode, equipmentCode:item.equipmentCode, standardHours:Number(item.hours) || 0 }
}))
const loadPlanPcgs = computed(() => ['전체', ...new Set(loadResources.value.map(item => item.pcg))])
const loadPlanPeriods = computed(() => loadPlanView.value === 'month'
  ? Array.from({ length:12 }, (_, index) => ({ key:index + 1, label:`${index + 1}월` }))
  : Array.from({ length:53 }, (_, index) => ({ key:index + 1, label:`${index + 1}주` })))
const filteredLoadResources = computed(() => {
  const keyword = loadPlanKeyword.value.trim().toLowerCase()
  return loadResources.value.filter(item =>
    (loadPlanPcg.value === '전체' || item.pcg === loadPlanPcg.value)
    && (!keyword || `${item.pcg} ${item.spec} ${item.code}`.toLowerCase().includes(keyword)))
})
function loadPlanHours(resource, period) {
  return Number(dispatchPlans.value.reduce((sum, plan) => {
    const detail = dispatchPlanDetail(plan)
    if (!detail || detail.rule.pcg !== resource.pcg || detail.rule.equipmentCode !== resource.equipmentCode) return sum
    const date = new Date(`${detail.dispatchDate}T00:00:00`)
    if (date.getFullYear() !== loadPlanYear.value) return sum
    const planPeriod = loadPlanView.value === 'month' ? date.getMonth() + 1 : getWeekNumber(date)
    return planPeriod === period ? sum + detail.requiredHours : sum
  }, 0).toFixed(1))
}
function loadPlanCellClass(value) {
  const capacity = loadPlanView.value === 'month' ? 80 : 20
  if (value >= capacity) return 'critical'
  if (value >= capacity * .7) return 'warning'
  return value > 0 ? 'normal' : 'empty'
}
const loadPlanTotalHours = computed(() => filteredLoadResources.value.reduce((sum, resource) =>
  sum + loadPlanPeriods.value.reduce((periodSum, period) => periodSum + loadPlanHours(resource, period.key), 0), 0))
const loadPlanPeriodTotals = computed(() => loadPlanPeriods.value.map(period => ({
  ...period,
  total:filteredLoadResources.value.reduce((sum, resource) => sum + loadPlanHours(resource, period.key), 0),
})))
const loadPlanPeak = computed(() => loadPlanPeriodTotals.value.reduce((peak, item) => item.total > peak.total ? item : peak, { label:'-', total:0 }))
const loadPlanOverloads = computed(() => filteredLoadResources.value.reduce((sum, resource) =>
  sum + loadPlanPeriods.value.filter(period => loadPlanCellClass(loadPlanHours(resource, period.key)) === 'critical').length, 0))
function formatPlanHours(value) {
  return new Intl.NumberFormat('ko-KR', { maximumFractionDigits:1 }).format(value)
}
function persistLoadMasters() {
  localStorage.setItem(LOAD_MASTER_STORAGE_KEY, JSON.stringify({
    equipment:equipmentMasters.value, standardTimes:standardTimeMasters.value, rules:dispatchRuleMasters.value,
  }))
}
function persistDispatchPlans() {
  localStorage.setItem(DISPATCH_PLAN_STORAGE_KEY, JSON.stringify(dispatchPlans.value))
}
function nextEntityId(prefix, items) {
  const max = items.reduce((value, item) => Math.max(value, Number(item.id.split('-').pop()) || 0), 0)
  return `${prefix}-${String(max + 1).padStart(3,'0')}`
}
function openMasterCreate() {
  masterEditingId.value = ''
  masterNotice.value = ''
  masterForm.value = masterTab.value === 'equipment'
    ? { code:'', name:'', category:'TP', spec:'', capacity:0, ownedCount:1, dailyHours:8, availability:90, enabled:true }
    : masterTab.value === 'time'
      ? { pcg:'', equipmentCode:equipmentMasters.value[0]?.code || '', hours:1 }
      : { pcg:'', stage:'중조', task:'', triggerStage:'중조', triggerPoint:'착수', offsetDays:0, equipmentCode:equipmentMasters.value[0]?.code || '', quantity:1 }
  masterEditorOpen.value = true
}
function editMaster(item) {
  masterEditingId.value = item.id
  masterNotice.value = ''
  masterForm.value = JSON.parse(JSON.stringify(item))
  masterEditorOpen.value = true
}
function saveMaster() {
  const form = JSON.parse(JSON.stringify(masterForm.value))
  if (masterTab.value === 'equipment') {
    if (!form.code.trim() || !form.name.trim() || !form.spec.trim()) return (masterNotice.value = '장비코드, 장비명, 규격을 입력해주세요.')
    const duplicate = equipmentMasters.value.some(item => item.code === form.code && item.id !== masterEditingId.value)
    if (duplicate) return (masterNotice.value = '이미 등록된 장비코드입니다.')
    form.id = masterEditingId.value || nextEntityId('EQ', equipmentMasters.value)
    const index = equipmentMasters.value.findIndex(item => item.id === form.id)
    index >= 0 ? equipmentMasters.value.splice(index,1,form) : equipmentMasters.value.push(form)
  } else if (masterTab.value === 'time') {
    if (!form.pcg.trim() || !form.equipmentCode || Number(form.hours) <= 0) return (masterNotice.value = 'PCG, 장비, 표준시간을 입력해주세요.')
    const duplicate = standardTimeMasters.value.some(item => item.pcg === form.pcg && item.equipmentCode === form.equipmentCode && item.id !== masterEditingId.value)
    if (duplicate) return (masterNotice.value = '같은 PCG와 장비의 표준시간이 이미 등록되어 있습니다.')
    form.id = masterEditingId.value || nextEntityId('ST', standardTimeMasters.value)
    const index = standardTimeMasters.value.findIndex(item => item.id === form.id)
    index >= 0 ? standardTimeMasters.value.splice(index,1,form) : standardTimeMasters.value.push(form)
  } else {
    if (!form.pcg.trim() || !form.task.trim() || !form.equipmentCode) return (masterNotice.value = 'PCG, 운송작업, 장비를 입력해주세요.')
    form.id = masterEditingId.value || nextEntityId('RL', dispatchRuleMasters.value)
    const index = dispatchRuleMasters.value.findIndex(item => item.id === form.id)
    index >= 0 ? dispatchRuleMasters.value.splice(index,1,form) : dispatchRuleMasters.value.push(form)
  }
  persistLoadMasters()
  masterEditorOpen.value = false
}
function deleteMaster(item) {
  masterNotice.value = ''
  if (masterTab.value === 'equipment') {
    if (standardTimeMasters.value.some(master => master.equipmentCode === item.code) || dispatchRuleMasters.value.some(rule => rule.equipmentCode === item.code)) return (masterNotice.value = '표준시간 또는 배차규칙에서 사용하는 장비는 삭제할 수 없습니다.')
    equipmentMasters.value = equipmentMasters.value.filter(master => master.id !== item.id)
  } else if (masterTab.value === 'time') {
    if (dispatchRuleMasters.value.some(rule => rule.pcg === item.pcg && rule.equipmentCode === item.equipmentCode)) return (masterNotice.value = '배차규칙에서 사용하는 표준시간은 삭제할 수 없습니다.')
    standardTimeMasters.value = standardTimeMasters.value.filter(master => master.id !== item.id)
  } else {
    if (dispatchPlans.value.some(plan => plan.ruleId === item.id)) return (masterNotice.value = '배차계획에서 사용하는 규칙은 삭제할 수 없습니다.')
    dispatchRuleMasters.value = dispatchRuleMasters.value.filter(master => master.id !== item.id)
  }
  persistLoadMasters()
}
function addDays(dateString, days) {
  const date = new Date(`${dateString}T00:00:00Z`)
  date.setUTCDate(date.getUTCDate() + Number(days || 0))
  return date.toISOString().slice(0,10)
}
function getWeekNumber(date) {
  const first = new Date(date.getFullYear(), 0, 1)
  return Math.ceil((((date - first) / 86400000) + first.getDay() + 1) / 7)
}
function dispatchPlanDetail(plan) {
  const rule = dispatchRuleMasters.value.find(item => item.id === plan.ruleId)
  if (!rule) return null
  const time = standardTimeMasters.value.find(item => item.pcg === rule.pcg && item.equipmentCode === rule.equipmentCode)
  const equipment = equipmentMasters.value.find(item => item.code === rule.equipmentCode)
  return {
    rule, equipment, standardHours:Number(time?.hours || 0),
    dispatchDate:addDays(plan.baseDate, rule.offsetDays),
    requiredHours:Number(((Number(rule.quantity) || 0) * (Number(time?.hours) || 0)).toFixed(1)),
  }
}
const filteredDispatchPlans = computed(() => dispatchPlans.value.filter(plan => {
  const detail = dispatchPlanDetail(plan)
  return detail && (dispatchStageFilter.value === '전체' || detail.rule.stage === dispatchStageFilter.value)
}))
function openDispatchCreate() {
  dispatchEditingId.value = ''
  dispatchNotice.value = ''
  dispatchForm.value = { project:'', block:'', ruleId:dispatchRuleMasters.value[0]?.id || '', baseDate:new Date().toISOString().slice(0,10), status:'계획' }
  dispatchEditorOpen.value = true
}
function editDispatch(plan) {
  dispatchEditingId.value = plan.id
  dispatchNotice.value = ''
  dispatchForm.value = JSON.parse(JSON.stringify(plan))
  dispatchEditorOpen.value = true
}
function saveDispatch() {
  const form = JSON.parse(JSON.stringify(dispatchForm.value))
  if (!form.project.trim() || !form.block.trim() || !form.ruleId || !form.baseDate) return (dispatchNotice.value = '호선, 블록, 배차규칙, 기준일을 입력해주세요.')
  form.id = dispatchEditingId.value || nextEntityId('DP', dispatchPlans.value)
  const index = dispatchPlans.value.findIndex(item => item.id === form.id)
  index >= 0 ? dispatchPlans.value.splice(index,1,form) : dispatchPlans.value.push(form)
  persistDispatchPlans()
  dispatchEditorOpen.value = false
}
function deleteDispatch(plan) {
  dispatchPlans.value = dispatchPlans.value.filter(item => item.id !== plan.id)
  persistDispatchPlans()
}
const now = computed(() => new Intl.DateTimeFormat('ko-KR', {
  year: 'numeric', month: '2-digit', day: '2-digit', weekday: 'short'
}).format(new Date()))

async function login() {
  loggedIn.value = true
  await nextTick()
  initializeVworldMap()
}
function isMenuOpen(label) {
  return openMenus.value.includes(label)
}
function toggleMenu(label) {
  openMenus.value = isMenuOpen(label)
    ? openMenus.value.filter(item => item !== label)
    : [...openMenus.value, label]
}
function menuContainsActive(menu) {
  return menu.label === activeMenu.value || menu.children?.some(menuContainsActive)
}
async function selectMenu(menu) {
  if (menu.children?.length) {
    if (collapsed.value) return selectMenuItem(menu.children[0].children?.[0]?.label || menu.children[0].label)
    toggleMenu(menu.label)
    return
  }
  return selectMenuItem(menu.label)
}
async function selectMenuItem(label) {
  activeMenu.value = label
  if (label === '물리지번 목록') {
    await nextTick()
    initializePhysicalMap()
  }
}
function mapScaleToLeafletZoom(scale = mapZoom.value) {
  return BASE_LEAFLET_ZOOM + Math.log2(scale)
}
function leafletZoomToMapScale(zoom) {
  return Math.min(MAX_MAP_ZOOM, Math.max(MIN_MAP_ZOOM, Number((2 ** (zoom - BASE_LEAFLET_ZOOM)).toFixed(2))))
}
function createVworldLeafletMap(element, showZoomControl = false) {
  return L.map(element, {
    zoomControl:showZoomControl,
    attributionControl:true,
    zoomSnap:.05,
    zoomDelta:.25,
    wheelPxPerZoomLevel:60,
    rotate:true,
    bearing:mapRotation.value,
    rotateControl:false,
    touchRotate:true,
    shiftKeyRotate:true,
  }).setView([34.9012707,127.593559], mapScaleToLeafletZoom())
}
function applyNativeMapView(map) {
  if (!map) return
  const targetZoom = mapScaleToLeafletZoom()
  if (Math.abs(map.getZoom() - targetZoom) > .01) map.setZoom(targetZoom, { animate:false })
  if (typeof map.setBearing === 'function' && Math.abs((map.getBearing?.() || 0) - mapRotation.value) > .1) map.setBearing(mapRotation.value)
}
function bindNativeMapView(map) {
  map.on('zoom', () => {
    const nextScale = leafletZoomToMapScale(map.getZoom())
    if (nextScale !== mapZoom.value) mapZoom.value = nextScale
  })
  map.on('rotate', () => {
    const bearing = Math.round(map.getBearing?.() || 0)
    if (bearing !== mapRotation.value) mapRotation.value = bearing
  })
}
function initializePhysicalMap() {
  if (!physicalMapElement.value || physicalMap) return
  physicalMap = createVworldLeafletMap(physicalMapElement.value)
  bindNativeMapView(physicalMap)
  physicalMap.on('click',handlePhysicalMapClick)
  physicalMap.on('mousemove',handlePhysicalMapMouseMove)
  physicalMap.on('contextmenu',handlePhysicalMapContextMenu)
  setPhysicalVworldLayer()
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
  if (drawMode.value) {
    physicalPolygonLayers = []
    return
  }
  const selectedParcel = physicalParcels.value.find(parcel => parcel.id === selectedPhysicalId.value)
  physicalPolygonLayers = selectedParcel?.points?.length
    ? [createParcelGridLayer(physicalMap, selectedParcel, { onClick:() => selectPhysicalParcel(selectedParcel.id) })]
    : []
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
  persistPhysicalParcels()
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
function physicalGridMetersFromLatLng(point) {
  const eastMeters = (point.lng - PHYSICAL_GRID_CENTER[1]) * METERS_PER_LONGITUDE_DEGREE
  const southMeters = (PHYSICAL_GRID_CENTER[0] - point.lat) * METERS_PER_LATITUDE_DEGREE
  const angle = mapRotation.value * Math.PI / 180
  const horizontalMeters = Math.cos(angle) * eastMeters - Math.sin(angle) * southMeters
  const verticalMeters = Math.sin(angle) * eastMeters + Math.cos(angle) * southMeters
  return { horizontal:horizontalMeters,vertical:verticalMeters }
}
function physicalGridPointFromMeters(horizontalMeters, verticalMeters) {
  const angle = mapRotation.value * Math.PI / 180
  const eastMeters = Math.cos(angle) * horizontalMeters + Math.sin(angle) * verticalMeters
  const southMeters = -Math.sin(angle) * horizontalMeters + Math.cos(angle) * verticalMeters
  return [
    PHYSICAL_GRID_CENTER[0] - southMeters / METERS_PER_LATITUDE_DEGREE,
    PHYSICAL_GRID_CENTER[1] + eastMeters / METERS_PER_LONGITUDE_DEGREE,
  ]
}
function interpolatePoint(start, end, ratio) {
  return [start[0] + (end[0] - start[0]) * ratio,start[1] + (end[1] - start[1]) * ratio]
}
function parcelGridSize(map, parcel) {
  const [topLeft,topRight,,bottomLeft] = parcel.points
  return {
    columns:parcel.gridColumns || Math.max(1,Math.min(100,Math.round(map.distance(topLeft,topRight) / PHYSICAL_GRID_CELL_METERS))),
    rows:parcel.gridRows || Math.max(1,Math.min(100,Math.round(map.distance(topLeft,bottomLeft) / PHYSICAL_GRID_CELL_METERS))),
  }
}
function createParcelGridLayer(map, parcel, { onClick, draft = false } = {}) {
  const [topLeft,topRight,bottomRight,bottomLeft] = parcel.points
  const { columns,rows } = parcelGridSize(map,parcel)
  const color = PHYSICAL_GRID_COLOR
  const group = L.layerGroup().addTo(map)
  const polygon = L.polygon(parcel.points, {
    color,weight:2,fillColor:color,fillOpacity:0,className:'physical-grid-outline',
    dashArray:parcel.enabled === false ? '5 5' : undefined,
  }).addTo(group)
  for (let column = 1; column < columns; column += 1) {
    const ratio = column / columns
    L.polyline([interpolatePoint(topLeft,topRight,ratio),interpolatePoint(bottomLeft,bottomRight,ratio)], { color,weight:1,opacity:.95,interactive:false,className:'physical-grid-line' }).addTo(group)
  }
  for (let row = 1; row < rows; row += 1) {
    const ratio = row / rows
    L.polyline([interpolatePoint(topLeft,bottomLeft,ratio),interpolatePoint(topRight,bottomRight,ratio)], { color,weight:1,opacity:.95,interactive:false,className:'physical-grid-line' }).addTo(group)
  }
  if (!draft) polygon.bindTooltip(`${parcel.id} · ${parcel.name}`, { permanent:true,direction:'center',className:'parcel-map-label' })
  if (onClick) polygon.on('click',onClick)
  return group
}
function renderPhysicalDraftGrid() {
  if (physicalDraftLayer && physicalMap) physicalMap.removeLayer(physicalDraftLayer)
  physicalDraftLayer = null
  if (!physicalMap || !physicalGridSelection.value) return
  const selection = physicalGridSelection.value
  physicalDraftLayer = createParcelGridLayer(physicalMap, {
    points:selection.points,
    gridColumns:selection.maxColumn - selection.minColumn + 1,
    gridRows:selection.maxRow - selection.minRow + 1,
    enabled:true,
  }, { draft:true })
}
function renderPhysicalGrid() {
  if (!physicalMap) return
  if (physicalGridLayer) physicalMap.removeLayer(physicalGridLayer)
  physicalGridLayer = null
}
function setPhysicalGridSelection(start, end) {
  const columns = Math.max(1,Math.ceil(Math.abs(end.horizontal - start.horizontal) / PHYSICAL_GRID_CELL_METERS))
  const rows = Math.max(1,Math.ceil(Math.abs(end.vertical - start.vertical) / PHYSICAL_GRID_CELL_METERS))
  const left = end.horizontal >= start.horizontal ? start.horizontal : start.horizontal - columns * PHYSICAL_GRID_CELL_METERS
  const right = left + columns * PHYSICAL_GRID_CELL_METERS
  const top = end.vertical >= start.vertical ? start.vertical : start.vertical - rows * PHYSICAL_GRID_CELL_METERS
  const bottom = top + rows * PHYSICAL_GRID_CELL_METERS
  const points = [
    physicalGridPointFromMeters(left,top),
    physicalGridPointFromMeters(right,top),
    physicalGridPointFromMeters(right,bottom),
    physicalGridPointFromMeters(left,bottom),
  ]
  physicalGridSelection.value = { minRow:0,maxRow:rows - 1,minColumn:0,maxColumn:columns - 1,points }
  if (physicalDraftParcel.value) {
    physicalDraftParcel.value.startParcel = '(0,0)'
    physicalDraftParcel.value.endParcel = `(${columns - 1},${rows - 1})`
  }
  renderPhysicalDraftGrid()
}
function handlePhysicalMapClick(event) {
  if (!drawMode.value) return
  const point = physicalGridMetersFromLatLng(event.latlng)
  if (!physicalGridDragStart) {
    physicalGridDragStart = point
    physicalGridAwaitingEnd.value = true
    setPhysicalGridSelection(point,point)
    return
  }
  setPhysicalGridSelection(physicalGridDragStart,point)
  physicalGridDragStart = null
  physicalGridAwaitingEnd.value = false
}
function clearPhysicalGridSelection(event) {
  if (!drawMode.value) return
  event.preventDefault()
  physicalGridDragStart = null
  physicalGridAwaitingEnd.value = false
  physicalGridSelection.value = null
  if (physicalDraftLayer && physicalMap) physicalMap.removeLayer(physicalDraftLayer)
  physicalDraftLayer = null
}
function handlePhysicalMapMouseMove(event) {
  if (!drawMode.value || !physicalGridDragStart) return
  setPhysicalGridSelection(physicalGridDragStart,physicalGridMetersFromLatLng(event.latlng))
}
function handlePhysicalMapContextMenu(event) {
  clearPhysicalGridSelection(event.originalEvent || event)
}
function clearPhysicalDraftLayers() {
  if (physicalDraftLayer && physicalMap) physicalMap.removeLayer(physicalDraftLayer)
  if (physicalGridLayer && physicalMap) physicalMap.removeLayer(physicalGridLayer)
  physicalDraftLayer = null
  physicalGridLayer = null
  physicalGridDragStart = null
  physicalGridAwaitingEnd.value = false
}
function startPhysicalDrawing() {
  if (drawMode.value) return
  const id = createPhysicalId()
  const today = currentKoreanDate()
  physicalParcels.value.push({ id, name:'새 물리지번', startParcel:'', endParcel:'', logicalParcel:'미연계', enabled:true, createdAt:today, createdBy:'관리자', updatedAt:today, updatedBy:'관리자', points:[], draft:true })
  editingPhysicalId.value = ''
  physicalEditSnapshot = undefined
  selectedPhysicalId.value = id
  drawMode.value = true
  physicalGridSelection.value = null
  physicalMap?.dragging.enable()
  renderPhysicalPolygons()
  renderPhysicalGrid()
}
function physicalSelectionFromParcel(parcel) {
  const { columns,rows } = parcelGridSize(physicalMap,parcel)
  return { minRow:0,maxRow:rows - 1,minColumn:0,maxColumn:columns - 1,points:parcel.points.map(point => [...point]) }
}
function startPhysicalEditing() {
  const parcel = selectedPhysicalParcel.value
  if (!parcel || drawMode.value || !physicalMap) return
  physicalEditSnapshot = JSON.parse(JSON.stringify(parcel))
  editingPhysicalId.value = parcel.id
  parcel.draft = true
  physicalGridSelection.value = physicalSelectionFromParcel(parcel)
  drawMode.value = true
  deleteConfirmPhysicalId.value = ''
  physicalMap.dragging.enable()
  renderPhysicalPolygons()
  renderPhysicalDraftGrid()
  physicalMap.fitBounds(parcel.points, { padding:[60,60],maxZoom:18 })
}
function resetPhysicalDrawing() {
  if (!drawMode.value || !physicalDraftParcel.value) return
  physicalGridDragStart = null
  physicalGridAwaitingEnd.value = false
  if (editingPhysicalId.value && physicalEditSnapshot) {
    Object.assign(physicalDraftParcel.value,JSON.parse(JSON.stringify(physicalEditSnapshot)),{ draft:true })
    physicalGridSelection.value = physicalSelectionFromParcel(physicalDraftParcel.value)
    renderPhysicalDraftGrid()
    return
  }
  physicalDraftParcel.value.name = '새 물리지번'
  physicalDraftParcel.value.enabled = true
  physicalGridSelection.value = null
  if (physicalDraftLayer && physicalMap) physicalMap.removeLayer(physicalDraftLayer)
  physicalDraftLayer = null
}
function cancelPhysicalDrawing() {
  if (editingPhysicalId.value && physicalEditSnapshot) {
    const parcel = physicalParcels.value.find(item => item.id === editingPhysicalId.value)
    if (parcel) {
      Object.assign(parcel,JSON.parse(JSON.stringify(physicalEditSnapshot)))
      delete parcel.draft
    }
    drawMode.value = false
    editingPhysicalId.value = ''
    physicalEditSnapshot = undefined
    physicalGridSelection.value = null
    clearPhysicalDraftLayers()
    physicalMap?.dragging.enable()
    renderPhysicalPolygons()
    return
  }
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
  draft.gridColumns = selection.maxColumn - selection.minColumn + 1
  draft.gridRows = selection.maxRow - selection.minRow + 1
  draft.updatedAt = currentKoreanDate()
  draft.updatedBy = '관리자'
  delete draft.draft
  drawMode.value = false
  editingPhysicalId.value = ''
  physicalEditSnapshot = undefined
  clearPhysicalDraftLayers()
  physicalGridSelection.value = null
  physicalMap?.dragging.enable()
  renderPhysicalPolygons()
  selectPhysicalParcel(draft.id)
  persistPhysicalParcels()
}
function requestPhysicalDelete() {
  if (!selectedPhysicalParcel.value) return
  deleteConfirmPhysicalId.value = selectedPhysicalParcel.value.id
}
function cancelPhysicalDelete() {
  deleteConfirmPhysicalId.value = ''
}
function confirmPhysicalDelete() {
  const id = deleteConfirmPhysicalId.value
  const index = physicalParcels.value.findIndex(parcel => parcel.id === id)
  if (index < 0) return
  splitMaps.value.filter(item => item.parcelId === id).forEach(item => removeSplitMap(item.id))
  physicalParcels.value.splice(index,1)
  deleteConfirmPhysicalId.value = ''
  selectedPhysicalId.value = physicalParcels.value[Math.min(index,physicalParcels.value.length - 1)]?.id || ''
  renderPhysicalPolygons()
  const selected = selectedPhysicalParcel.value
  if (selected?.points?.length) physicalMap?.fitBounds(selected.points,{ padding:[60,60],maxZoom:18 })
  persistPhysicalParcels()
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
  mapZoom.value = Math.min(MAX_MAP_ZOOM, Math.max(MIN_MAP_ZOOM, Number((mapZoom.value + delta).toFixed(2))))
}
function handleMapWheel(event) {
  zoomMap(event.deltaY < 0 ? .05 : -.05)
}
function applyZoomInput(event) {
  const value = Number(event.target.value)
  const percent = Number.isFinite(value) ? Math.min(200, Math.max(MIN_MAP_ZOOM * 100, Math.round(value))) : 100
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
function pointInsideParcel(parcel,u,v) {
  const [topLeft,topRight,bottomRight,bottomLeft] = parcel.points
  const top = [topLeft[0] + (topRight[0] - topLeft[0]) * u,topLeft[1] + (topRight[1] - topLeft[1]) * u]
  const bottom = [bottomLeft[0] + (bottomRight[0] - bottomLeft[0]) * u,bottomLeft[1] + (bottomRight[1] - bottomLeft[1]) * u]
  return [top[0] + (bottom[0] - top[0]) * v,top[1] + (bottom[1] - top[1]) * v]
}
function dashboardMarkerIcon(item) {
  const typeClass = { 자재:'material',블록:'block',방해요소:'obstacle',운송자원:'transport',주요시설:'facility' }[item.type]
  const focusedClass = focusedCode.value === item.code ? ' focused' : ''
  return L.divIcon({
    className:'dashboard-marker-shell',
    html:`<span class="dashboard-map-marker dashboard-map-marker--${typeClass}${focusedClass}">${item.label}</span>`,
    iconSize:[34,34],iconAnchor:[17,17],
  })
}
function renderDashboardObjects() {
  if (!vworldMap) return
  dashboardObjectLayer?.remove()
  dashboardSiteLayer?.remove()
  dashboardObjectLayer = L.layerGroup().addTo(vworldMap)
  const parcel = dashboardSiteParcel.value
  if (!parcel?.points?.length) return
  dashboardSiteLayer = L.polygon(parcel.points,{ color:'#ed7100',weight:2,opacity:.9,fill:false,dashArray:'7 5',interactive:false,className:'dashboard-site-boundary' }).addTo(vworldMap)
  const visibleObjects = dashboardObjects.filter(item => displayItems.value.includes(item.type))
  if (!visibleObjects.length) return
  if (vworldMap.getZoom() < 15.6) {
    const counts = visibleObjects.reduce((result,item) => ({ ...result,[item.type]:(result[item.type] || 0) + 1 }),{})
    const legend = Object.entries(counts).map(([type,count]) => `<span>${type} <b>${count}</b></span>`).join('')
    const cluster = L.marker(pointInsideParcel(parcel,.5,.5),{
      icon:L.divIcon({ className:'dashboard-cluster-shell',html:`<div class="dashboard-object-cluster"><strong>${visibleObjects.length}</strong><small>표시 객체</small><div>${legend}</div></div>`,iconSize:[150,112],iconAnchor:[75,56] }),
      keyboard:true,title:`한화오션에코텍 표시 객체 ${visibleObjects.length}개`,alt:`한화오션에코텍 표시 객체 ${visibleObjects.length}개`,
    })
    cluster.addTo(dashboardObjectLayer)
    return
  }
  visibleObjects.forEach(item => {
    const marker = L.marker(pointInsideParcel(parcel,item.u,item.v),{
      icon:dashboardMarkerIcon(item),keyboard:true,title:`${item.code} · ${item.type}`,alt:`${item.code} ${item.type}`,
    })
    if (objectDetails[item.code]) marker.on('click',() => showObjectDetails(item.code))
    marker.addTo(dashboardObjectLayer)
  })
}
function initializeVworldMap() {
  if (!vworldKey || !vworldMapElement.value || vworldMap) return
  vworldMap = createVworldLeafletMap(vworldMapElement.value)
  bindNativeMapView(vworldMap)
  vworldMap.on('zoomend moveend',renderDashboardObjects)
  setVworldLayer()
  renderDashboardObjects()
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
  layers.push(createParcelGridLayer(map,parcel))
  splitLeafletLayers.set(item.id, layers)
}
function setSplitMapType(item, type) {
  item.mapType = type
  refreshSplitMapLayers(item)
}
function zoomSplitMap(item, delta) {
  item.zoom = Math.min(2, Math.max(MIN_MAP_ZOOM, Number((item.zoom + delta).toFixed(2))))
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
  dashboardObjectLayer = undefined
  dashboardSiteLayer = undefined
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
watch(displayItems,renderDashboardObjects,{ deep:true })
watch(focusedCode,renderDashboardObjects)
watch(mapRotation, () => {
  if (!drawMode.value) return
  physicalGridSelection.value = null
  if (physicalDraftLayer && physicalMap) physicalMap.removeLayer(physicalDraftLayer)
  physicalDraftLayer = null
  renderPhysicalGrid()
})
watch(activeMenu, async (menu, previousMenu) => {
  if (previousMenu === '대시보드' && menu !== '대시보드') destroyVworldMap()
  if (previousMenu === '물리지번 목록' && menu !== '물리지번 목록') destroyPhysicalMap()

  await nextTick()
  if (menu === '대시보드') initializeVworldMap()
  if (menu === '물리지번 목록') initializePhysicalMap()
})
watch([mapZoom, mapRotation], ([zoom, rotation]) => {
  localStorage.setItem('hanwha-map-view', JSON.stringify({ zoom, rotation }))
  applyNativeMapView(vworldMap)
  applyNativeMapView(physicalMap)
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
          <button
            :class="{ active: menuContainsActive(menu) }"
            :title="collapsed ? menu.label : undefined"
            :aria-label="menu.label"
            :aria-expanded="menu.children ? isMenuOpen(menu.label) : undefined"
            @click="selectMenu(menu)"
          >
            <component :is="menu.icon" class="menu-icon" aria-hidden="true" />
            <span v-if="!collapsed">{{ menu.label }}</span>
            <span v-if="menu.children && !collapsed" class="submenu-arrow">{{ isMenuOpen(menu.label) ? '⌃' : '⌄' }}</span>
          </button>
          <div v-if="menu.children && isMenuOpen(menu.label) && !collapsed" class="submenu-list">
            <template v-for="submenu in menu.children" :key="submenu.label">
              <button
                :class="{ active: menuContainsActive(submenu), 'has-children': submenu.children }"
                :aria-expanded="submenu.children ? isMenuOpen(submenu.label) : undefined"
                @click.stop="submenu.children ? toggleMenu(submenu.label) : selectMenuItem(submenu.label)"
              >
                <span>{{ submenu.label }}</span>
                <span v-if="submenu.children" class="submenu-arrow">{{ isMenuOpen(submenu.label) ? '⌃' : '⌄' }}</span>
              </button>
              <div v-if="submenu.children && isMenuOpen(submenu.label)" class="submenu-list submenu-list--third">
                <button
                  v-for="thirdMenu in submenu.children"
                  :key="thirdMenu.label"
                  :class="{ active: activeMenu === thirdMenu.label }"
                  @click.stop="selectMenuItem(thirdMenu.label)"
                >
                  {{ thirdMenu.label }}
                </button>
              </div>
            </template>
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
            <div ref="mapContainer" class="map-stage" :class="{ 'has-vworld': vworldReady }">
            <div ref="vworldMapElement" class="vworld-map"></div>
            <div class="mock-map" :style="{ transform: `rotate(${mapRotation}deg)`, '--counter-rotation': `${-mapRotation}deg` }">
              <div v-if="mapOptions.includes('CAD 도면')" class="cad-overlay"><i></i><i></i><i></i><i></i></div>
            </div>
              <div v-if="!vworldKey" class="vworld-key-notice"><strong>VWorld 지도 준비됨</strong><span>API 인증키를 등록하면 실제 지도가 표시됩니다.</span></div>
              <form class="map-search" role="search" @submit.prevent="focusMapObject">
                <label for="object-code">객체 코드 검색</label>
                <div><input id="object-code" v-model="searchCode" placeholder="예: MAT-12, G1" autocomplete="off" @input="searchMessage = ''" /><button type="submit">검색</button><button v-if="focusedCode" type="button" class="clear-search" aria-label="검색 포커스 해제" @click="clearObjectFocus">×</button></div>
                <p v-if="searchMessage" :class="{ error: !focusedCode }">{{ searchMessage }}</p>
              </form>
              <div class="map-view-controls" aria-label="지도 보기 제어">
                <button title="확대" aria-label="지도 확대" :disabled="mapZoom >= 2" @click="zoomMap(.25)">＋</button>
                <label class="zoom-input"><input :value="Math.round(mapZoom * 100)" type="number" min="40" max="200" step="5" aria-label="지도 확대 비율" @change="applyZoomInput" /><span>%</span></label>
                <button title="축소" aria-label="지도 축소" :disabled="mapZoom <= MIN_MAP_ZOOM" @click="zoomMap(-.25)">−</button>
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
                      <button aria-label="서브 지도 축소" :disabled="subMap.zoom <= MIN_MAP_ZOOM" @click="zoomSplitMap(subMap, -.25)">−</button>
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
        <header class="physical-page-header"><div><h1>물리지번 목록</h1><p>야드의 실제 공간 영역을 등록하고 표시 순서를 관리합니다.</p></div><div class="physical-actions"><button v-if="!drawMode" class="outline-button" @click="startPhysicalDrawing">＋ 물리지번 추가</button><template v-else><span class="draw-guide">클릭으로 시작점과 끝점을 지정하고, 드래그로 지도를 이동하세요.</span><button class="gray-button" @click="resetPhysicalDrawing">초기화</button><button class="gray-button" @click="cancelPhysicalDrawing">취소</button></template></div></header>
        <div class="physical-layout">
          <aside class="physical-list-panel">
            <div class="panel-heading"><div><h2>물리지번</h2><span>{{ physicalParcels.length }}개</span></div><small>끌어서 표시 순서를 변경합니다.</small></div>
            <div class="physical-list">
              <template v-for="(parcel,index) in physicalParcels" :key="parcel.id">
                <div v-if="parcel.draft" class="physical-list-draft selected" draggable="true" @dragstart="dragPhysicalStart(index)" @dragover.prevent @drop="dropPhysicalAt(index)">
                  <span class="drag-handle" title="순서 변경">⠿</span><span class="draft-folder" aria-hidden="true">▰</span><label><span class="sr-only">물리지번 명칭</span><input v-model="parcel.name" maxlength="40" placeholder="물리지번 명칭 입력" /></label><b>{{ editingPhysicalId === parcel.id ? '수정중' : '작성중' }}</b>
                </div>
                <button v-else draggable="true" :class="{ selected: selectedPhysicalId === parcel.id, dragging: draggingPhysicalIndex === index }" :aria-pressed="selectedPhysicalId === parcel.id" @dragstart="dragPhysicalStart(index)" @dragover.prevent @drop="dropPhysicalAt(index)" @click="selectPhysicalParcel(parcel.id)">
                  <span class="drag-handle" title="순서 변경">⠿</span><i :style="{ background: PHYSICAL_GRID_COLOR }"></i><span><strong>{{ parcel.name }}</strong><small>{{ parcel.id }} · {{ parcel.logicalParcel }}</small></span><b :class="{ off: !parcel.enabled }">{{ parcel.enabled ? '사용' : '미사용' }}</b>
                </button>
              </template>
            </div>
          </aside>
          <section ref="physicalMapContainer" class="physical-map-panel">
            <div ref="physicalMapElement" class="physical-map"></div>
            <div class="physical-map-types segmented" aria-label="지도 유형"><button v-for="type in mapTypes" :key="type.value" :class="{ selected: mapType === type.value }" @click="mapType = type.value">{{ type.label }}</button></div>
            <div class="physical-map-controls map-view-controls" aria-label="지도 보기 제어">
              <button title="확대" aria-label="지도 확대" :disabled="mapZoom >= 2" @click="zoomMap(.25)">＋</button>
              <label class="zoom-input"><input :value="Math.round(mapZoom * 100)" type="number" min="40" max="200" step="5" aria-label="지도 확대 비율" @change="applyZoomInput" /><span>%</span></label>
              <button title="축소" aria-label="지도 축소" :disabled="mapZoom <= MIN_MAP_ZOOM" @click="zoomMap(-.25)">−</button>
              <button title="왼쪽으로 15도 회전" aria-label="지도 반시계 방향 15도 회전" @click="rotateMap(-1)">↺</button>
              <button title="오른쪽으로 15도 회전" aria-label="지도 시계 방향 15도 회전" @click="rotateMap(1)">↻</button>
              <button title="원위치" aria-label="지도 보기 원위치" @click="resetMapView">⌂</button>
              <button title="전체화면" aria-label="지도 전체화면 전환" @click="togglePhysicalFullscreen">⛶</button>
            </div>
            <label class="physical-rotation rotation-indicator">회전 <input v-model.number="mapRotation" type="number" min="-180" max="180" step="1" aria-label="지도 회전 각도" @change="applyRotationInput" /><span>°</span></label>
            <div v-if="drawMode" class="drawing-badge">{{ physicalGridAwaitingEnd ? '끝점을 클릭하세요' : '시작점을 클릭하세요' }} · 드래그로 지도 이동</div>
          </section>
          <aside v-if="drawMode && physicalDraftParcel" class="physical-detail-panel physical-create-panel">
            <div class="detail-heading"><span>{{ editingPhysicalId ? '물리지번 수정' : '신규 등록' }}</span><h2>{{ physicalDraftParcel.name || '물리지번 명칭 미입력' }}</h2><small>{{ editingPhysicalId ? '기존 정보를 유지한 상태에서 명칭, 사용 여부와 격자 영역을 수정할 수 있습니다.' : '격자 영역을 지정하면 좌표가 자동 계산됩니다.' }}</small></div>
            <dl class="physical-create-fields">
              <div><dt>ID</dt><dd><input :value="physicalDraftParcel.id" readonly aria-label="자동 생성 ID" /></dd></div>
              <div><dt>시작 지번</dt><dd><input :value="physicalGridAddresses.start" readonly placeholder="영역 지정 필요" aria-label="시작 지번" /></dd></div>
              <div><dt>끝 지번</dt><dd><input :value="physicalGridAddresses.end" readonly placeholder="영역 지정 필요" aria-label="끝 지번" /></dd></div>
              <div><dt>사용여부</dt><dd class="physical-radio-group"><label><input v-model="physicalDraftParcel.enabled" type="radio" :value="true" /> 사용</label><label><input v-model="physicalDraftParcel.enabled" type="radio" :value="false" /> 미사용</label></dd></div>
              <div><dt>생성일자</dt><dd><input :value="physicalDraftParcel.createdAt" readonly aria-label="생성일자" /></dd></div>
              <div><dt>생성자</dt><dd><input :value="physicalDraftParcel.createdBy" readonly aria-label="생성자" /></dd></div>
            </dl>
            <div class="physical-selection-summary" aria-live="polite"><strong v-if="physicalGridSelection">{{ physicalGridSelection.maxColumn - physicalGridSelection.minColumn + 1 }} × {{ physicalGridSelection.maxRow - physicalGridSelection.minRow + 1 }}칸 선택</strong><span v-if="physicalGridSelection">실제 크기 {{ (physicalGridSelection.maxColumn - physicalGridSelection.minColumn + 1) * 10 }}m × {{ (physicalGridSelection.maxRow - physicalGridSelection.minRow + 1) * 10 }}m</span><span v-else>지도에서 시작점을 클릭해주세요.</span></div>
            <div class="detail-footer"><button class="primary-small" :disabled="!physicalGridSelection || physicalGridAwaitingEnd || !physicalDraftParcel.name.trim()" @click="completePhysicalDrawing">{{ editingPhysicalId ? '수정 완료' : '완료' }}</button></div>
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
            <div v-if="deleteConfirmPhysicalId === selectedPhysicalParcel.id" class="physical-delete-confirm" role="alert"><p><strong>{{ selectedPhysicalParcel.name }}</strong> 물리지번을 삭제하시겠습니까?</p><div><button class="gray-button" @click="cancelPhysicalDelete">취소</button><button class="danger-button" @click="confirmPhysicalDelete">삭제 확인</button></div></div>
            <div v-else class="detail-footer"><button class="gray-button" @click="requestPhysicalDelete">삭제</button><button class="primary-small" @click="startPhysicalEditing">수정</button></div>
          </aside>
        </div>
      </div>

      <div v-else-if="activeMenu === '부하율마스터관리'" class="management-page">
        <header class="management-header">
          <div><h1>부하율 마스터 관리</h1><p>자동배차와 부하율 계산에 사용하는 기준정보를 관리합니다.</p></div>
          <button class="primary-small" type="button" @click="openMasterCreate">＋ {{ masterTab === 'rule' ? '배차규칙' : masterTab === 'equipment' ? '장비' : '표준시간' }} 등록</button>
        </header>
        <nav class="master-tabs" aria-label="부하율 마스터 유형">
          <button :class="{ active: masterTab === 'rule' }" @click="masterTab = 'rule'; masterNotice = ''"><strong>운송작업 규칙</strong><span>{{ dispatchRuleMasters.length }}건</span><small>생산단계별 장비 투입시점</small></button>
          <button :class="{ active: masterTab === 'equipment' }" @click="masterTab = 'equipment'; masterNotice = ''"><strong>장비 마스터</strong><span>{{ equipmentMasters.length }}건</span><small>TP·크레인 규격과 가용시간</small></button>
          <button :class="{ active: masterTab === 'time' }" @click="masterTab = 'time'; masterNotice = ''"><strong>표준시간 마스터</strong><span>{{ standardTimeMasters.length }}건</span><small>PCG·장비별 기준 작업시간</small></button>
        </nav>
        <p v-if="masterNotice" class="management-notice" role="alert">{{ masterNotice }}</p>
        <section class="management-card">
          <table v-if="masterTab === 'rule'" class="management-table">
            <thead><tr><th>ID</th><th>PCG</th><th>생산단계</th><th>운송작업</th><th>기준절점</th><th>상대일수</th><th>장비</th><th>수량</th><th>관리</th></tr></thead>
            <tbody><tr v-for="item in dispatchRuleMasters" :key="item.id"><td>{{ item.id }}</td><td>{{ item.pcg }}</td><td>{{ item.stage }}</td><td>{{ item.task }}</td><td>{{ item.triggerStage }} {{ item.triggerPoint }}</td><td>{{ Number(item.offsetDays) > 0 ? '+' : '' }}{{ item.offsetDays }}일</td><td>{{ item.equipmentCode }}</td><td>{{ item.quantity }}대</td><td class="row-actions"><button @click="editMaster(item)">수정</button><button class="delete" @click="deleteMaster(item)">삭제</button></td></tr></tbody>
          </table>
          <table v-else-if="masterTab === 'equipment'" class="management-table">
            <thead><tr><th>ID</th><th>장비코드</th><th>장비명</th><th>구분</th><th>규격</th><th>보유대수</th><th>일 가동시간</th><th>가동률</th><th>사용여부</th><th>관리</th></tr></thead>
            <tbody><tr v-for="item in equipmentMasters" :key="item.id"><td>{{ item.id }}</td><td><strong>{{ item.code }}</strong></td><td>{{ item.name }}</td><td>{{ item.category }}</td><td>{{ item.spec }}</td><td>{{ item.ownedCount }}대</td><td>{{ item.dailyHours }}HR</td><td>{{ item.availability }}%</td><td><span class="use-chip" :class="{ off:!item.enabled }">{{ item.enabled ? '사용' : '미사용' }}</span></td><td class="row-actions"><button @click="editMaster(item)">수정</button><button class="delete" @click="deleteMaster(item)">삭제</button></td></tr></tbody>
          </table>
          <table v-else class="management-table">
            <thead><tr><th>ID</th><th>PCG</th><th>장비코드</th><th>장비명</th><th>표준시간</th><th>관리</th></tr></thead>
            <tbody><tr v-for="item in standardTimeMasters" :key="item.id"><td>{{ item.id }}</td><td>{{ item.pcg }}</td><td><strong>{{ item.equipmentCode }}</strong></td><td>{{ equipmentMasters.find(master => master.code === item.equipmentCode)?.name }}</td><td>{{ item.hours }} HR</td><td class="row-actions"><button @click="editMaster(item)">수정</button><button class="delete" @click="deleteMaster(item)">삭제</button></td></tr></tbody>
          </table>
        </section>
        <div v-if="masterEditorOpen" class="editor-backdrop" @click.self="masterEditorOpen = false">
          <section class="editor-dialog" role="dialog" aria-modal="true" aria-labelledby="master-editor-title">
            <header><div><span>{{ masterEditingId ? '기준정보 수정' : '신규 기준정보' }}</span><h2 id="master-editor-title">{{ masterTab === 'rule' ? '운송작업 규칙' : masterTab === 'equipment' ? '장비 마스터' : '표준시간 마스터' }}</h2></div><button aria-label="등록창 닫기" @click="masterEditorOpen = false">×</button></header>
            <div v-if="masterTab === 'equipment'" class="editor-fields">
              <label>장비코드<input v-model.trim="masterForm.code" placeholder="예: TP032" /></label><label>장비명<input v-model.trim="masterForm.name" placeholder="예: 32톤 TP" /></label>
              <label>장비구분<select v-model="masterForm.category"><option>TP</option><option>크레인</option><option>지게차</option></select></label><label>규격<input v-model.trim="masterForm.spec" placeholder="예: 032" /></label>
              <label>용량(톤)<input v-model.number="masterForm.capacity" type="number" min="0" /></label><label>보유대수<input v-model.number="masterForm.ownedCount" type="number" min="1" /></label>
              <label>일 가동시간<input v-model.number="masterForm.dailyHours" type="number" min="1" max="24" /></label><label>가동률(%)<input v-model.number="masterForm.availability" type="number" min="0" max="100" /></label>
              <label class="editor-checkbox"><input v-model="masterForm.enabled" type="checkbox" /> 사용 장비</label>
            </div>
            <div v-else-if="masterTab === 'time'" class="editor-fields">
              <label>PCG<input v-model.trim="masterForm.pcg" placeholder="예: LNGC DK" /></label><label>장비<select v-model="masterForm.equipmentCode"><option v-for="item in equipmentMasters" :key="item.id" :value="item.code">{{ item.code }} · {{ item.name }}</option></select></label>
              <label>표준시간(HR)<input v-model.number="masterForm.hours" type="number" min=".1" step=".1" /></label>
            </div>
            <div v-else class="editor-fields">
              <label>PCG<input v-model.trim="masterForm.pcg" placeholder="예: LNGC DK" /></label><label>생산단계<select v-model="masterForm.stage"><option v-for="stage in ['소조','중조','대조','도장','PE','선적']" :key="stage">{{ stage }}</option></select></label>
              <label class="wide">운송작업<input v-model.trim="masterForm.task" placeholder="예: 대조주판 배치" /></label><label>기준단계<select v-model="masterForm.triggerStage"><option v-for="stage in ['소조','중조','대조','도장','PE','선적']" :key="stage">{{ stage }}</option></select></label>
              <label>기준절점<select v-model="masterForm.triggerPoint"><option>착수</option><option>완료</option></select></label><label>상대일수<input v-model.number="masterForm.offsetDays" type="number" /></label>
              <label>투입장비<select v-model="masterForm.equipmentCode"><option v-for="item in equipmentMasters.filter(master => master.enabled)" :key="item.id" :value="item.code">{{ item.code }} · {{ item.name }}</option></select></label><label>투입수량<input v-model.number="masterForm.quantity" type="number" min="1" /></label>
            </div>
            <p v-if="masterNotice" class="editor-error">{{ masterNotice }}</p>
            <footer><button class="gray-button" @click="masterEditorOpen = false">취소</button><button class="primary-small" @click="saveMaster">{{ masterEditingId ? '수정 완료' : '등록' }}</button></footer>
          </section>
        </div>
      </div>

      <div v-else-if="activeMenu === '중기배차계획'" class="management-page">
        <header class="management-header"><div><h1>중기배차계획</h1><p>생산계획의 기준일과 배차규칙을 연결해 장비 투입일과 필요시간을 자동 계산합니다.</p></div><button class="primary-small" @click="openDispatchCreate">＋ 배차계획 등록</button></header>
        <ol class="workflow-strip" aria-label="부하율 업무 흐름"><li class="done"><span>1</span><strong>마스터 관리</strong><small>장비·규칙·표준시간</small></li><li class="active"><span>2</span><strong>중기배차계획</strong><small>생산단계별 투입계획</small></li><li><span>3</span><strong>부하율 분석</strong><small>주·월 통계와 과부하</small></li></ol>
        <section class="dispatch-toolbar"><label>생산단계<select v-model="dispatchStageFilter"><option>전체</option><option v-for="stage in ['소조','중조','대조','도장','PE','선적']" :key="stage">{{ stage }}</option></select></label><span>총 {{ filteredDispatchPlans.length }}건</span></section>
        <section class="management-card">
          <table class="management-table dispatch-table"><thead><tr><th>계획ID</th><th>호선</th><th>블록</th><th>PCG</th><th>생산단계</th><th>운송작업</th><th>기준일</th><th>투입예정일</th><th>장비</th><th>필요시간</th><th>상태</th><th>관리</th></tr></thead>
            <tbody><tr v-for="plan in filteredDispatchPlans" :key="plan.id"><td>{{ plan.id }}</td><td><strong>{{ plan.project }}</strong></td><td>{{ plan.block }}</td><td>{{ dispatchPlanDetail(plan)?.rule.pcg }}</td><td>{{ dispatchPlanDetail(plan)?.rule.stage }}</td><td>{{ dispatchPlanDetail(plan)?.rule.task }}</td><td>{{ plan.baseDate }}</td><td><strong class="orange-text">{{ dispatchPlanDetail(plan)?.dispatchDate }}</strong></td><td>{{ dispatchPlanDetail(plan)?.rule.equipmentCode }} × {{ dispatchPlanDetail(plan)?.rule.quantity }}</td><td>{{ dispatchPlanDetail(plan)?.requiredHours }} HR</td><td><span class="plan-status" :class="plan.status">{{ plan.status }}</span></td><td class="row-actions"><button @click="editDispatch(plan)">수정</button><button class="delete" @click="deleteDispatch(plan)">삭제</button></td></tr></tbody>
          </table>
        </section>
        <div v-if="dispatchEditorOpen" class="editor-backdrop" @click.self="dispatchEditorOpen = false">
          <section class="editor-dialog" role="dialog" aria-modal="true" aria-labelledby="dispatch-editor-title">
            <header><div><span>{{ dispatchEditingId ? '배차계획 수정' : '신규 계획' }}</span><h2 id="dispatch-editor-title">생산단계별 중기배차계획</h2></div><button aria-label="배차계획 등록창 닫기" @click="dispatchEditorOpen = false">×</button></header>
            <div class="editor-fields"><label>호선<input v-model.trim="dispatchForm.project" placeholder="예: HNOE-3101" /></label><label>블록<input v-model.trim="dispatchForm.block" placeholder="예: A12-01" /></label>
              <label class="wide">배차규칙<select v-model="dispatchForm.ruleId"><option v-for="rule in dispatchRuleMasters" :key="rule.id" :value="rule.id">{{ rule.pcg }} · {{ rule.stage }} · {{ rule.task }} · {{ rule.equipmentCode }}</option></select></label>
              <label>생산계획 기준일<input v-model="dispatchForm.baseDate" type="date" /></label><label>상태<select v-model="dispatchForm.status"><option>계획</option><option>확정</option><option>완료</option></select></label>
            </div>
            <div v-if="dispatchPlanDetail(dispatchForm)" class="dispatch-preview"><span>자동 계산 결과</span><strong>{{ dispatchPlanDetail(dispatchForm).dispatchDate }} · {{ dispatchPlanDetail(dispatchForm).rule.equipmentCode }} {{ dispatchPlanDetail(dispatchForm).rule.quantity }}대 · {{ dispatchPlanDetail(dispatchForm).requiredHours }} HR</strong></div>
            <p v-if="dispatchNotice" class="editor-error">{{ dispatchNotice }}</p><footer><button class="gray-button" @click="dispatchEditorOpen = false">취소</button><button class="primary-small" @click="saveDispatch">{{ dispatchEditingId ? '수정 완료' : '계획 등록' }}</button></footer>
          </section>
        </div>
      </div>

      <div v-else-if="activeMenu === '운송자원별 부하율 분석/통계'" class="load-plan-page">
        <header class="load-plan-header">
          <div><h1>TP 주·월별 부하 계획</h1><p>생산계획과 장비 표준시간을 기준으로 TP 규격별 예상 투입시간을 조회합니다.</p></div>
          <div class="load-plan-view" role="group" aria-label="조회 단위">
            <button :class="{ active: loadPlanView === 'week' }" @click="loadPlanView = 'week'">주별</button>
            <button :class="{ active: loadPlanView === 'month' }" @click="loadPlanView = 'month'">월별</button>
          </div>
        </header>

        <section class="load-plan-filters" aria-label="부하 계획 조회 조건">
          <label>기준연도<select v-model.number="loadPlanYear"><option :value="2025">2025년</option><option :value="2026">2026년</option><option :value="2027">2027년</option></select></label>
          <label>PCG<select v-model="loadPlanPcg"><option v-for="pcg in loadPlanPcgs" :key="pcg">{{ pcg }}</option></select></label>
          <label class="load-plan-search">장비 검색<input v-model="loadPlanKeyword" type="search" placeholder="PCG, 규격, 장비코드" /></label>
          <button class="gray-button" type="button" @click="loadPlanPcg = '전체'; loadPlanKeyword = ''">초기화</button>
        </section>

        <section class="load-plan-summary" aria-label="부하 계획 요약">
          <article><span>조회 장비</span><strong>{{ filteredLoadResources.length }}<small>종</small></strong><p>TP 규격·PCG 조합</p></article>
          <article><span>총 투입시간</span><strong>{{ formatPlanHours(loadPlanTotalHours) }}<small>HR</small></strong><p>{{ loadPlanView === 'month' ? '월별' : '주별' }} 계획 합계</p></article>
          <article><span>최대 부하 구간</span><strong>{{ loadPlanPeak.label }}</strong><p>{{ formatPlanHours(loadPlanPeak.total) }} HR 예정</p></article>
          <article class="risk"><span>과부하 예상</span><strong>{{ loadPlanOverloads }}<small>건</small></strong><p>가용시간 기준 초과</p></article>
        </section>

        <section class="load-plan-card">
          <div class="load-plan-card-head">
            <div><h2>{{ loadPlanYear }}년 TP {{ loadPlanView === 'month' ? '월별' : '주별' }} 투입시간</h2><p>단위: HR · 셀을 가로로 스크롤하여 전체 기간을 확인할 수 있습니다.</p></div>
            <div class="load-legend" aria-label="부하 상태 범례"><span><i class="normal"></i>정상</span><span><i class="warning"></i>주의</span><span><i class="critical"></i>과부하</span></div>
          </div>
          <div class="load-plan-table-wrap" tabindex="0" aria-label="TP 부하 계획표 가로 스크롤 영역">
            <table class="load-plan-table">
              <caption class="sr-only">TP 장비별 {{ loadPlanView === 'month' ? '월간' : '주간' }} 예상 투입시간</caption>
              <thead><tr><th scope="col">TP 규격</th><th scope="col">PCG</th><th scope="col">장비코드</th><th scope="col">표준시간</th><th scope="col">합계</th><th v-for="period in loadPlanPeriods" :key="period.key" scope="col">{{ period.label }}</th></tr></thead>
              <tbody>
                <tr v-for="resource in filteredLoadResources" :key="resource.code">
                  <th scope="row">{{ resource.spec }}</th><td>{{ resource.pcg }}</td><td><strong>{{ resource.code }}</strong></td><td>{{ resource.standardHours }} HR</td>
                  <td class="row-total">{{ formatPlanHours(loadPlanPeriods.reduce((sum, period) => sum + loadPlanHours(resource, period.key), 0)) }}</td>
                  <td v-for="period in loadPlanPeriods" :key="period.key" :class="loadPlanCellClass(loadPlanHours(resource, period.key))">{{ formatPlanHours(loadPlanHours(resource, period.key)) }}</td>
                </tr>
                <tr v-if="!filteredLoadResources.length"><td :colspan="loadPlanPeriods.length + 5" class="load-plan-empty">조회 조건에 해당하는 장비가 없습니다.</td></tr>
              </tbody>
              <tfoot v-if="filteredLoadResources.length"><tr><th colspan="4" scope="row">기간 합계</th><td>{{ formatPlanHours(loadPlanTotalHours) }}</td><td v-for="period in loadPlanPeriodTotals" :key="period.key">{{ formatPlanHours(period.total) }}</td></tr></tfoot>
            </table>
          </div>
        </section>
      </div>

      <div v-else class="empty-page"><span class="empty-icon">▦</span><h1>{{ activeMenu }}</h1><p>이 메뉴는 다음 단계에서 구성할 예정입니다.</p><button class="outline-button" @click="activeMenu = '대시보드'">대시보드로 이동</button></div>
    </main>
  </div>
</template>
