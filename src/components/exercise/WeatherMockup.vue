<script setup>
import { ref } from 'vue'

// 1. 배열 렌더링 (v-for)
const weatherList = ref([
  { id: 'city_01', name: '서울', temp: 28, status: '맑음' },
  { id: 'city_02', name: '수원', temp: 24, status: '비' },
  { id: 'city_03', name: '부산', temp: 26, status: '구름' },
])

// 2. 조건부 렌더링 (v-if)
</script>

<template>
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
