<script setup>
import { ref } from 'vue'

const weatherList = ref([
  { id: 'city_01', name: '서울', temp: 28, status: '맑음' },
  { id: 'city_02', name: '수원', temp: 24, status: '비' },
  { id: 'city_03', name: '부산', temp: 26, status: '구름' },
])

const searchQuery = ref('')
</script>

<template>
  <section class="search-box">
    <h3>🔍 도시 검색</h3>
    <input
      type="text"
      :value="searchQuery"
      @input="(e) => (searchQuery = e.target.value)"
      placeholder="검색할 도시 이름 입력"
    />
    <p>
      검색 중인 도시: <strong>{{ searchQuery }}</strong>
    </p>
  </section>
  <section class="list-box">
    <h3>날씨 현황</h3>
    <div v-for="item in weatherList" :key="item.id" class="weather-card">
      <h4>{{ item.name }} ({{ item.status }})</h4>
      <p>현재 기온: {{ item.temp }}°C</p>

      <span v-if="item.temp >= 25" class="badge hot">🔥 더움 (25도 이상)</span>
      <span v-else class="badge cool">❄️ 선선함 (25도 미만)</span>
    </div>
  </section>
</template>

<style></style>
