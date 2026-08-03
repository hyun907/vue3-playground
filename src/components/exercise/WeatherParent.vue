<script setup>
import { ref, computed, watch, watchEffect, onMounted, onUnmounted } from 'vue'
import { useRouter } from 'vue-router'
import axios from 'axios'
// 1. 컴포넌트 파일명 국룰 표기법(PascalCase) 매칭 수입
import BaseDashboardCard from './BaseDashboardCard.vue'
import SearchBar from './SearchBar.vue'
import WeatherCard from './WeatherCard.vue'

// 실제 외부 API 데이터를 받아와 채워줄 빈 반응형 배열과 로딩/에러 상태
const weatherList = ref([])
const isLoading = ref(false)
const hasError = ref(false)

const searchQuery = ref('')
const selectedCityInfo = ref('카드를 클릭하거나 검색해 보세요.')

// OpenWeatherMap 연동 규격 — 키는 .env의 VITE_OPENWEATHER_API_KEY로 관리
const API_KEY = import.meta.env.VITE_OPENWEATHER_API_KEY
const BASE_URL = 'https://api.openweathermap.org/data/2.5/weather'
// 지오코딩 API: 날씨 API와 달리 한글 도시명("대구")도 좌표로 변환해준다
const GEO_URL = 'https://api.openweathermap.org/geo/1.0/direct'

// 처음 방문했을 때 띄워줄 기본 도시 목록
const BASE_CITIES = [
  { query: 'Seoul', name: '서울' },
  { query: 'Suwon', name: '수원' },
  { query: 'Busan', name: '부산' },
]

// 브라우저에 목록을 남겨둘 localStorage 키
const STORAGE_KEY = 'skala-weather-cities'

// 저장된 목록 읽기 — 값이 없으면 null(기본 도시 사용), 빈 배열이면 "모두 지운 상태"로 존중한다
const loadSavedCities = () => {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (raw === null) return null
    const parsed = JSON.parse(raw)
    return Array.isArray(parsed) ? parsed : null
  } catch (error) {
    console.error('🔴 저장된 도시 목록을 읽지 못했습니다:', error)
    return null
  }
}

// API 원본 응답을 카드가 쓰는 규격으로 정규화 (id는 OpenWeatherMap 도시 코드)
// timezone은 UTC 기준 오프셋(초) — 카드에서 현지 시각을 계산하는 데 쓴다
const toCityCard = (raw, koreanName) => ({
  id: raw.id,
  name: koreanName,
  temp: Math.round(raw.main.temp),
  status: raw.weather[0].description,
  timezone: raw.timezone,
})

// 대상 도시들의 실시간 날씨를 병렬(Promise.all)로 수신해 반응형 장부에 동기화.
// 기온·상태는 저장하지 않고 매번 새로 받아오므로 항상 최신값이 표시된다.
const fetchRealTimeWeather = async (targets = null) => {
  const cities = targets ?? BASE_CITIES
  isLoading.value = true
  hasError.value = false
  try {
    if (cities.length === 0) {
      weatherList.value = []
      return
    }

    const responses = await Promise.all(
      cities.map((city) =>
        axios.get(BASE_URL, {
          params: {
            // 저장된 도시는 도시 코드로, 기본 도시는 영문명으로 조회
            ...(city.id != null ? { id: city.id } : { q: city.query }),
            appid: API_KEY,
            units: 'metric',
            lang: 'kr',
          },
        }),
      ),
    )

    weatherList.value = responses.map((res, idx) => toCityCard(res.data, cities[idx].name))
    console.log('🟢 [API 통신 완료] 실시간 기상 장부 동기화:', weatherList.value)
  } catch (error) {
    console.error('🔴 날씨 API 연동 실패:', error)
    hasError.value = true
  } finally {
    isLoading.value = false
  }
}

// 저장 기록을 지우고 기본 도시 3종으로 되돌린다
const resetToDefault = () => {
  localStorage.removeItem(STORAGE_KEY)
  fetchRealTimeWeather()
}

// ── 목록에 없는 도시 원격 검색 ────────────────────────────
const remoteCity = ref(null)
const isSearching = ref(false)
const searchNotFound = ref(false)
let searchTimer = null

// 지오코딩으로 좌표를 먼저 얻고, 그 좌표로 실시간 날씨를 조회하는 2단 요청
const searchRemoteCity = async (query) => {
  isSearching.value = true
  searchNotFound.value = false
  remoteCity.value = null
  try {
    const geoRes = await axios.get(GEO_URL, {
      params: { q: query, limit: 1, appid: API_KEY },
    })
    const place = geoRes.data[0]
    if (!place) {
      searchNotFound.value = true
      return
    }

    const weatherRes = await axios.get(BASE_URL, {
      params: { lat: place.lat, lon: place.lon, appid: API_KEY, units: 'metric', lang: 'kr' },
    })
    // 현지 표기(local_names.ko)가 있으면 한글 지명을 우선 사용
    remoteCity.value = toCityCard(weatherRes.data, place.local_names?.ko || place.name)
    console.log('🔎 [원격 검색 성공]', remoteCity.value)
  } catch (error) {
    console.error('🔴 도시 검색 실패:', error)
    searchNotFound.value = true
  } finally {
    isSearching.value = false
  }
}

// 카드마다 현지 시각을 흘려보내기 위한 공용 시계 (1초마다 갱신)
const now = ref(Date.now())
let clockTimer = null

// 컴포넌트가 화면에 장착(mounted)되는 시점에, 저장해둔 목록이 있으면 그 목록으로 복원
onMounted(() => {
  fetchRealTimeWeather(loadSavedCities())
  clockTimer = setInterval(() => (now.value = Date.now()), 1000)
})

// 목록이 바뀔 때마다 도시 신원(코드·한글명)만 브라우저에 저장
watch(
  weatherList,
  (list) => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(list.map(({ id, name }) => ({ id, name }))))
  },
  { deep: true },
)

// 기존 핵심 비즈니스 로직(computed, watch)의 소유권은 안전하게 부모 콘텍스트가 격리 유지
const filteredWeatherList = computed(() => {
  const query = searchQuery.value.trim()
  if (!query) return weatherList.value
  return weatherList.value.filter((item) => item.name.includes(query))
})

// 기본 목록에 없는 검색어면 디바운스 후 원격 검색으로 넘긴다
watch(searchQuery, (value) => {
  clearTimeout(searchTimer)
  remoteCity.value = null
  searchNotFound.value = false
  isSearching.value = false

  const query = value.trim()
  if (!query || filteredWeatherList.value.length > 0) return

  isSearching.value = true
  searchTimer = setTimeout(() => searchRemoteCity(query), 500)
})

// 화면에서 내려갈 때 예약된 타이머 정리
onUnmounted(() => {
  clearTimeout(searchTimer)
  clearInterval(clockTimer)
})

// 기본 목록 우선, 비어 있으면 원격 검색 결과로 대체
const displayList = computed(() => {
  if (filteredWeatherList.value.length > 0) return filteredWeatherList.value
  return remoteCity.value ? [remoteCity.value] : []
})

// 지금 보여주는 카드가 저장된 목록이 아니라 원격 검색 결과인지
const isRemoteResult = computed(
  () => filteredWeatherList.value.length === 0 && remoteCity.value !== null,
)

// ── 목록 추가 / 제거 ──────────────────────────────────
const addCity = (city) => {
  if (weatherList.value.some((item) => item.id === city.id)) {
    selectedCityInfo.value = `${city.name} 카드는 이미 목록에 있습니다.`
    return
  }
  weatherList.value.push(city)
  selectedCityInfo.value = `${city.name} 카드를 목록에 추가했습니다.`
  // 검색어를 비우면 watch가 원격 검색 상태까지 정리하고 전체 목록으로 되돌아간다
  searchQuery.value = ''
}

const removeCity = (city) => {
  weatherList.value = weatherList.value.filter((item) => item.id !== city.id)
  selectedCityInfo.value = `${city.name} 카드를 목록에서 제거했습니다.`
}

watch(selectedCityInfo, (newInfo) => {
  console.log(`👁️‍🗨️ [watch 감지] 상태 바 문구가 업데이트되었습니다 -> "${newInfo}"`)
})

watchEffect(() => {
  console.log(
    `🤖 [watchEffect 자동 호출] 현재 검색어 '${searchQuery.value}'에 매칭되는 API 데이터를 필터링합니다.`,
  )
})

// 상세보기 클릭 시 라우터로 동적 파라미터(/weather/:cityId) 상세 페이지 이동
// 한글 지명은 API가 돌려주지 않으므로 쿼리로 함께 넘긴다
const router = useRouter()
const goDetail = (city) => {
  router.push({ name: 'WeatherDetail', params: { cityId: city.id }, query: { name: city.name } })
}
</script>

<template>
  <div class="dashboard-wrapper">
    <aside class="dash-side">
      <BaseDashboardCard style="--index: 1">
        <SearchBar :current-query="searchQuery" @update-query="(val) => (searchQuery = val)" />
      </BaseDashboardCard>

      <div class="status-bar">
        <span class="status-dot"></span>
        {{ selectedCityInfo }}
      </div>
    </aside>

    <BaseDashboardCard class="dash-list" style="--index: 2">
      <div class="section-head">
        <h3>지역별 날씨 현황</h3>
        <span class="count-chip">{{ displayList.length }}개 도시</span>
      </div>

      <div v-if="isLoading" class="loading-state">
        <span class="loading-glyph">🛰️</span>
        <p>실시간 기상 데이터를 수신 중입니다...</p>
      </div>

      <div v-else-if="hasError" class="error-state">
        <span class="error-glyph">🔴</span>
        <p>API 호출이 실패했습니다. (API 키 한도 초과이거나 네트워크 문제일 수 있습니다)</p>
        <button class="retry-btn" @click="fetchRealTimeWeather">다시 시도</button>
      </div>

      <template v-else>
        <p v-if="isRemoteResult" class="remote-hint">
          🔎 목록에 없는 도시입니다. <strong>추가</strong> 버튼으로 목록에 담을 수 있어요.
        </p>

        <div class="card-grid">
          <WeatherCard
            v-for="(item, idx) in displayList"
            :key="item.id"
            :city-item="item"
            :now="now"
            :addable="isRemoteResult"
            :removable="!isRemoteResult"
            :style="{ '--index': idx }"
            @select-card="(msg) => (selectedCityInfo = msg)"
            @click-detail="goDetail(item)"
            @add-card="addCity(item)"
            @remove-card="removeCity(item)"
          />
        </div>

        <div v-if="isSearching" class="loading-state">
          <span class="loading-glyph">🔎</span>
          <p>'{{ searchQuery }}' 도시를 전 세계에서 찾는 중입니다...</p>
        </div>

        <div v-else-if="searchNotFound" class="empty-state">
          <span class="empty-glyph">🌫️</span>
          <p>'{{ searchQuery }}' 이름의 도시를 찾지 못했습니다.</p>
          <p class="empty-tip">도시 이름을 한글 또는 영문으로 정확히 입력해 보세요.</p>
        </div>

        <div v-else-if="displayList.length === 0" class="empty-state">
          <span class="empty-glyph">🗂️</span>
          <p>목록에 담긴 도시가 없습니다.</p>
          <p class="empty-tip">위 검색창에서 도시를 찾아 추가해 보세요.</p>
          <button class="retry-btn" @click="resetToDefault">기본 도시 다시 불러오기</button>
        </div>
      </template>
    </BaseDashboardCard>
  </div>
</template>

<style scoped>
/* PC: 좌측 고정 사이드바(검색·상태) + 우측 카드 그리드 투컬럼 */
.dashboard-wrapper {
  width: 100%;
  display: grid;
  grid-template-columns: 340px minmax(0, 1fr);
  gap: 0 28px;
  align-items: start;
}

.dash-side {
  position: sticky;
  top: 32px;
}

.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 12px;
}

/* 900px 이하: 사이드바를 해체해 검색 → 리스트 → 상태 바 순서의 단일 컬럼으로 */
@media (max-width: 900px) {
  .dashboard-wrapper {
    grid-template-columns: 1fr;
  }
  .dash-side {
    display: contents;
  }
  .dash-side .status-bar {
    order: 3;
  }
}

.section-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 18px;
}
.section-head h3 {
  margin: 0;
  font-size: 15px;
  font-weight: 700;
  letter-spacing: -0.01em;
  color: var(--sky-ink);
}
.count-chip {
  padding: 4px 12px;
  border-radius: 999px;
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.06em;
  color: var(--sky-accent);
  background: rgba(125, 211, 252, 0.08);
  border: 1px solid rgba(125, 211, 252, 0.16);
}

.loading-state,
.error-state,
.empty-state {
  text-align: center;
  padding: 36px 0 28px;
  color: var(--sky-ink-dim);
}
.loading-glyph,
.error-glyph {
  display: block;
  font-size: 34px;
  margin-bottom: 10px;
}
.loading-glyph {
  animation: pulse 1.6s var(--sky-ease) infinite;
}
.loading-state p,
.error-state p {
  margin: 0;
  font-size: 14px;
}
.error-state p {
  color: #f87171;
}
.empty-tip {
  margin-top: 8px !important;
  font-size: 12px;
  opacity: 0.7;
}
.remote-hint strong {
  font-weight: 700;
  color: var(--sky-ink);
}
.remote-hint {
  margin: 0 0 14px;
  padding: 8px 14px;
  border-radius: 999px;
  font-size: 12px;
  text-align: center;
  color: var(--sky-accent);
  background: rgba(125, 211, 252, 0.07);
  border: 1px solid rgba(125, 211, 252, 0.16);
}
.retry-btn {
  margin-top: 14px;
  padding: 8px 20px;
  border-radius: 999px;
  font-size: 13px;
  font-weight: 600;
  color: var(--sky-ink);
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(255, 255, 255, 0.12);
  cursor: pointer;
  transition: background 0.3s var(--sky-ease);
}
.retry-btn:hover {
  background: rgba(255, 255, 255, 0.12);
}
.empty-glyph {
  display: block;
  font-size: 34px;
  margin-bottom: 10px;
}
.empty-state p {
  margin: 0;
  font-size: 14px;
}

.status-bar {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  padding: 14px 20px;
  border-radius: 999px;
  font-size: 14px;
  font-weight: 500;
  color: var(--sky-ink);
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.08);
  box-shadow: inset 0 1px 1px rgba(255, 255, 255, 0.08);
  animation: cardRise 0.8s var(--sky-ease) both;
  animation-delay: 360ms;
}
.status-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--sky-accent);
  box-shadow: 0 0 10px rgba(125, 211, 252, 0.8);
  animation: pulse 2.4s var(--sky-ease) infinite;
}
@keyframes pulse {
  0%,
  100% {
    opacity: 1;
    transform: scale(1);
  }
  50% {
    opacity: 0.45;
    transform: scale(0.8);
  }
}

@keyframes cardRise {
  from {
    opacity: 0;
    transform: translateY(2rem);
    filter: blur(4px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
    filter: blur(0);
  }
}
</style>
