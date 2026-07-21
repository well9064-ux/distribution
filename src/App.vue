<script setup>
import { computed, nextTick, onBeforeUnmount, ref, watch } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

const loggedIn = ref(false)
const collapsed = ref(false)
const activeMenu = ref('대시보드')
const showPassword = ref(false)
const loginId = ref('')
const password = ref('')
const mapType = ref('일반')
const mapOptions = ref(['차도 색상', '지번 표시', '적치 여부'])
const displayItems = ref(['자재', '블록', '운송자원', '주요시설'])
const mapZoom = ref(1)
const mapRotation = ref(0)
const mapContainer = ref(null)
const vworldMapElement = ref(null)
const vworldReady = ref(false)
const vworldKey = import.meta.env.VITE_VWORLD_API_KEY || ''
let vworldMap
let vworldLayer
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
  { label: '대시보드', icon: '▦' },
  { label: '지번관리', icon: '⌖' },
  { label: '입고예정정보', icon: '⇥' },
  { label: '블록관리', icon: '⬡' },
  { label: '운송수단관리', icon: '▰' },
  { label: '시스템관리', icon: '⚙' },
]

const now = computed(() => new Intl.DateTimeFormat('ko-KR', {
  year: 'numeric', month: '2-digit', day: '2-digit', weekday: 'short'
}).format(new Date()))

async function login() {
  loggedIn.value = true
  await nextTick()
  initializeVworldMap()
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
  mapType.value = '일반'
  mapOptions.value = ['차도 색상', '지번 표시', '적치 여부']
  displayItems.value = ['자재', '블록', '운송자원', '주요시설']
  resetMapView()
}
function zoomMap(delta) {
  mapZoom.value = Math.min(2, Math.max(.75, Number((mapZoom.value + delta).toFixed(2))))
}
function rotateMap() { mapRotation.value = (mapRotation.value + 45) % 360 }
function resetMapView() { mapZoom.value = 1; mapRotation.value = 0 }
async function toggleFullscreen() {
  if (!document.fullscreenElement) await mapContainer.value?.requestFullscreen()
  else await document.exitFullscreen()
}
function vworldTileUrl(type) {
  const layer = type === '위성' ? 'Satellite' : 'Base'
  const extension = type === '위성' ? 'jpeg' : 'png'
  return `https://api.vworld.kr/req/wmts/1.0.0/${vworldKey}/${layer}/{z}/{y}/{x}.${extension}`
}
function setVworldLayer() {
  if (!vworldMap || !vworldKey) return
  if (vworldLayer) vworldMap.removeLayer(vworldLayer)
  vworldLayer = L.tileLayer(vworldTileUrl(mapType.value), {
    minZoom: 6, maxZoom: 19,
    attribution: '&copy; VWorld 공간정보 오픈플랫폼'
  }).addTo(vworldMap)
  vworldReady.value = true
}
function initializeVworldMap() {
  if (!vworldKey || !vworldMapElement.value || vworldMap) return
  vworldMap = L.map(vworldMapElement.value, { zoomControl: false, attributionControl: true }).setView([34.88, 128.68], 16)
  setVworldLayer()
}
function focusMapObject() {
  const code = searchCode.value.trim().toUpperCase().replace(/\s+/g, '')
  const objects = {
    'A-12': '블록', 'A12': '블록', 'B-04': '블록', 'B04': '블록', 'C-08': '블록', 'C08': '블록',
    'MAT-12': '자재', 'MAT12': '자재', 'MAT-08': '자재', 'MAT08': '자재',
    'F-03': '운송자원', 'F03': '운송자원', 'CR-02': '운송자원', 'CR02': '운송자원', 'TP-07': '운송자원', 'TP07': '운송자원',
    'OBS-01': '방해요소', 'OBS01': '방해요소', 'OBS-02': '방해요소', 'OBS02': '방해요소',
    'G1': '주요시설', 'WH': '주요시설'
  }
  const canonical = { A12: 'A-12', B04: 'B-04', C08: 'C-08', MAT12: 'MAT-12', MAT08: 'MAT-08', F03: 'F-03', CR02: 'CR-02', TP07: 'TP-07', OBS01: 'OBS-01', OBS02: 'OBS-02' }
  if (!objects[code]) {
    focusedCode.value = ''
    searchMessage.value = '일치하는 객체 코드가 없습니다.'
    return
  }
  const item = objects[code]
  if (!displayItems.value.includes(item)) displayItems.value = [...displayItems.value, item]
  if (item === '블록' && !mapOptions.value.includes('지번 표시')) mapOptions.value = [...mapOptions.value, '지번 표시']
  focusedCode.value = canonical[code] || code
  searchCode.value = focusedCode.value
  searchMessage.value = `${focusedCode.value} 객체를 찾았습니다.`
  mapZoom.value = 1.35
  mapRotation.value = 0
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
watch(mapType, setVworldLayer)
onBeforeUnmount(() => vworldMap?.remove())
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
          <span aria-hidden="true">◯</span>
          <input id="login-id" v-model="loginId" autocomplete="username" placeholder="아이디를 입력해주세요" />
        </div>

        <label for="login-password">비밀번호</label>
        <div class="field-wrap">
          <span aria-hidden="true">⌑</span>
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
        <div class="brand-lockup">
          <span class="brand-mark"><i></i><i></i><i></i></span>
          <span v-if="!collapsed">Hanwha</span>
        </div>
        <button class="collapse-button" :aria-label="collapsed ? '메뉴 펼치기' : '메뉴 접기'" @click="collapsed = !collapsed">{{ collapsed ? '›' : '‹' }}</button>
      </div>
      <nav class="menu-list" aria-label="주 메뉴">
        <button v-for="menu in menus" :key="menu.label" :class="{ active: activeMenu === menu.label }" :title="collapsed ? menu.label : undefined" @click="activeMenu = menu.label">
          <span class="menu-icon">{{ menu.icon }}</span><span v-if="!collapsed">{{ menu.label }}</span>
        </button>
      </nav>
      <div class="sidebar-footer">
        <div class="avatar">관</div>
        <div v-if="!collapsed"><strong>관리자</strong><small>시스템 관리자</small></div>
        <button v-if="!collapsed" class="logout" @click="loggedIn = false" aria-label="로그아웃">↪</button>
      </div>
    </aside>

    <section class="workspace">
      <header class="topbar">
        <div><span class="home-icon">◆</span><span>›</span><strong>{{ activeMenu }}</strong></div>
        <div class="topbar-meta"><span>{{ now }}</span><button aria-label="알림">♢<i></i></button><span class="user-chip">관</span><strong>관리자</strong></div>
      </header>

      <div v-if="activeMenu === '대시보드'" class="dashboard">
        <div class="page-heading"><div><h1>대시보드</h1><p>야드 운영 현황을 실시간으로 확인합니다.</p></div><span class="live"><i></i> 실시간 업데이트</span></div>

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
              <button v-for="type in ['일반지도','위성지도']" :key="type" :class="{ selected: mapType === type.replace('지도', '') }" @click="mapType = type.replace('지도', '')">{{ type }}</button>
            </div>
          </div>
          <div class="map-area">
            <div ref="mapContainer" class="map-stage" :class="{ 'has-vworld': vworldReady }">
            <div ref="vworldMapElement" class="vworld-map" :style="{ transform: `scale(${mapZoom}) rotate(${mapRotation}deg)` }"></div>
            <div class="mock-map" :class="`map-${mapType}`" :style="{ transform: `scale(${mapZoom}) rotate(${mapRotation}deg)` }">
              <div class="water"><span>거제만</span></div>
              <div v-if="mapOptions.includes('CAD 도면')" class="cad-overlay"><i></i><i></i><i></i><i></i></div>
              <button v-if="displayItems.includes('블록')" type="button" class="yard-block block-a object-button" :class="{ focused: focusedCode === 'A-12' }" @click="showObjectDetails('A-12')"><b><span v-if="mapOptions.includes('지번 표시')">A-12 · </span>BLOCK</b><small v-if="mapOptions.includes('적치 여부')">적치중 · 82%</small></button>
              <button v-if="displayItems.includes('블록')" type="button" class="yard-block block-b object-button" :class="{ focused: focusedCode === 'B-04' }" @click="showObjectDetails('B-04')"><b><span v-if="mapOptions.includes('지번 표시')">B-04 · </span>BLOCK</b><small v-if="mapOptions.includes('적치 여부')">적치중 · 64%</small></button>
              <button v-if="displayItems.includes('블록')" type="button" class="yard-block block-c object-button" :class="{ focused: focusedCode === 'C-08' }" @click="showObjectDetails('C-08')"><b><span v-if="mapOptions.includes('지번 표시')">C-08 · </span>BLOCK</b><small v-if="mapOptions.includes('적치 여부')">미적치 · 0%</small></button>
              <div class="road road-one" :class="{ 'road-colored': mapOptions.includes('차도 색상') }"></div><div class="road road-two" :class="{ 'road-colored': mapOptions.includes('차도 색상') }"></div>
              <template v-if="displayItems.includes('운송자원')"><button type="button" class="map-object machine forklift m1" :class="{ focused: focusedCode === 'F-03' }" title="F-03 · 지게차" @click="showObjectDetails('F-03')"><span class="vehicle-icon">🚜</span><small>F-03</small></button><button type="button" class="map-object machine crane m2" :class="{ focused: focusedCode === 'CR-02' }" title="CR-02 · 크레인" @click="showObjectDetails('CR-02')"><span class="vehicle-icon">🏗</span><small>CR-02</small></button><button type="button" class="map-object machine transporter m3" :class="{ focused: focusedCode === 'TP-07' }" title="TP-07 · TP" @click="showObjectDetails('TP-07')"><span class="vehicle-icon">▰</span><small>TP-07</small></button></template>
              <template v-if="displayItems.includes('자재')"><button type="button" class="map-object material ma1" :class="{ focused: focusedCode === 'MAT-12' }" title="MAT-12 · 입고 자재" @click="showObjectDetails('MAT-12')">12</button><button type="button" class="map-object material ma2" :class="{ focused: focusedCode === 'MAT-08' }" title="MAT-08 · 입고 자재" @click="showObjectDetails('MAT-08')">8</button></template>
              <template v-if="displayItems.includes('방해요소')"><span class="map-object obstacle ob1" :class="{ focused: focusedCode === 'OBS-01' }" title="OBS-01 · 통행 방해요소">!</span><span class="map-object obstacle ob2" :class="{ focused: focusedCode === 'OBS-02' }" title="OBS-02 · 통행 방해요소">!</span></template>
              <template v-if="displayItems.includes('주요시설')"><span class="map-object facility fa1" :class="{ focused: focusedCode === 'G1' }" title="G1 · 제1 게이트">G1</span><span class="map-object facility fa2" :class="{ focused: focusedCode === 'WH' }" title="WH · 자재 창고">WH</span></template>
              <div class="map-label">한화오션에코텍 야드</div>
            </div>
              <div v-if="!vworldKey" class="vworld-key-notice"><strong>VWorld 지도 준비됨</strong><span>API 인증키를 등록하면 실제 지도가 표시됩니다.</span></div>
              <form class="map-search" role="search" @submit.prevent="focusMapObject">
                <label for="object-code">객체 코드 검색</label>
                <div><input id="object-code" v-model="searchCode" placeholder="예: A-12, MAT-12" autocomplete="off" @input="searchMessage = ''" /><button type="submit">검색</button><button v-if="focusedCode" type="button" class="clear-search" aria-label="검색 포커스 해제" @click="clearObjectFocus">×</button></div>
                <p v-if="searchMessage" :class="{ error: !focusedCode }">{{ searchMessage }}</p>
              </form>
              <div class="map-view-controls" aria-label="지도 보기 제어">
                <button title="확대" aria-label="지도 확대" :disabled="mapZoom >= 2" @click="zoomMap(.25)">＋</button>
                <span>{{ Math.round(mapZoom * 100) }}%</span>
                <button title="축소" aria-label="지도 축소" :disabled="mapZoom <= .75" @click="zoomMap(-.25)">−</button>
                <button title="회전" aria-label="지도 시계 방향 45도 회전" @click="rotateMap">↻</button>
                <button title="원위치" aria-label="지도 보기 원위치" @click="resetMapView">⌂</button>
                <button title="전체화면" aria-label="지도 전체화면 전환" @click="toggleFullscreen">⛶</button>
              </div>
              <span class="rotation-indicator">{{ mapRotation }}°</span>
              <aside v-if="selectedObject" class="object-detail" aria-live="polite">
                <header><div><span>{{ selectedObject.type }}</span><h3>{{ selectedObject.title }} 상세정보</h3></div><button type="button" aria-label="상세정보 닫기" @click="selectedObject = null">×</button></header>
                <dl><div v-for="row in selectedObject.rows" :key="row[0]"><dt>{{ row[0] }}</dt><dd :class="{ status: row[0] === '현재상태' || row[0] === '작업현황' }">{{ row[1] }}</dd></div></dl>
              </aside>
            </div>
            <aside class="map-options">
              <section><h3>지도 옵션</h3><button class="text-button" @click="resetMapSettings">초기화</button><label v-for="option in ['차도 색상','CAD 도면','지번 표시','적치 여부']" :key="option" class="switch-row"><span><i class="option-icon"></i>{{ option }}</span><input type="checkbox" :checked="mapOptions.includes(option)" @change="toggleOption(option)" /></label></section>
              <section><h3>표시 항목</h3><label v-for="item in ['자재','블록','방해요소','운송자원','주요시설']" :key="item" class="switch-row"><span><i :class="`legend legend-${item}`"></i>{{ item }}</span><input type="checkbox" :checked="displayItems.includes(item)" @change="toggleItem(item)" /></label></section>
              <section class="legend-section"><h3>점유율 범례</h3><div><span><i class="occ low"></i>0–40%</span><span><i class="occ mid"></i>41–70%</span><span><i class="occ high"></i>71–100%</span></div></section>
            </aside>
          </div>
        </section>
      </div>

      <div v-else class="empty-page"><span class="empty-icon">▦</span><h1>{{ activeMenu }}</h1><p>이 메뉴는 다음 단계에서 구성할 예정입니다.</p><button class="outline-button" @click="activeMenu = '대시보드'">대시보드로 이동</button></div>
    </section>
  </div>
</template>
