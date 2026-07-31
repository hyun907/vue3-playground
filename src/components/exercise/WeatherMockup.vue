<script setup>
import { ref } from 'vue'

// 1. 배열 렌더링 (v-for)
const weatherList = ref([
  { id: 'city_01', name: '서울', temp: 28, status: '맑음' },
  { id: 'city_02', name: '수원', temp: 24, status: '비' },
  { id: 'city_03', name: '부산', temp: 26, status: '구름' },
])
</script>

<template>
  <section class="list-box">
    <h3>날씨 현황</h3>
    <div v-for="item in weatherList" :key="item.id" class="weather-card">
      <h4>{{ item.name }} ({{ item.status }})</h4>
      <p>현재 기온: {{ item.temp }}°C</p>
    </div>
  </section>
</template>

<style></style>
