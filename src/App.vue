<script setup>
import { RouterView } from 'vue-router'
</script>

<template>
  <div class="sky-shell">
    <!-- 배경 장식: 메시 그라디언트 오브 -->
    <div class="orb orb-a" aria-hidden="true"></div>
    <div class="orb orb-b" aria-hidden="true"></div>
    <div class="orb orb-c" aria-hidden="true"></div>
    <div class="grain" aria-hidden="true"></div>

    <main class="sky-main">
      <header class="hero">
        <span class="eyebrow">Weather Intelligence</span>
        <h1 class="hero-title">
          오늘의 하늘을<br />
          <em>한눈에</em> 읽어드립니다
        </h1>
        <p class="hero-sub">도시를 검색하고 카드를 눌러 실시간 날씨 흐름을 확인하세요.</p>
      </header>

      <RouterView />
    </main>
  </div>
</template>

<style>
@import url('https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/variable/pretendardvariable-dynamic-subset.min.css');

:root {
  --sky-ease: cubic-bezier(0.16, 1, 0.3, 1);
  --sky-accent: #7dd3fc;
  --sky-accent-deep: #818cf8;
  --sky-ink: #f4f6fb;
  --sky-ink-dim: rgba(244, 246, 251, 0.55);
}

/* 기본 템플릿(main.css)의 레이아웃 제약 해제 */
body {
  display: block !important;
  background: #050505;
  min-height: 100dvh;
}
#app {
  max-width: none !important;
  padding: 0 !important;
  display: block !important;
  font-weight: normal;
}

.sky-shell {
  position: relative;
  min-height: 100dvh;
  overflow: hidden;
  background:
    radial-gradient(1200px 800px at 85% -10%, rgba(129, 140, 248, 0.14), transparent 60%),
    radial-gradient(1000px 700px at -10% 30%, rgba(125, 211, 252, 0.1), transparent 55%),
    #050505;
  font-family: 'Pretendard Variable', Pretendard, -apple-system, BlinkMacSystemFont, system-ui, sans-serif;
  color: var(--sky-ink);
  word-break: keep-all;
}

/* ── 떠다니는 배경 오브 ───────────────────────── */
.orb {
  position: absolute;
  border-radius: 50%;
  filter: blur(90px);
  opacity: 0.5;
  pointer-events: none;
  animation: orbFloat 9s var(--sky-ease) infinite;
}
.orb-a {
  width: 420px;
  height: 420px;
  top: -140px;
  right: -80px;
  background: radial-gradient(circle, rgba(129, 140, 248, 0.35), transparent 70%);
}
.orb-b {
  width: 360px;
  height: 360px;
  bottom: 5%;
  left: -120px;
  background: radial-gradient(circle, rgba(125, 211, 252, 0.28), transparent 70%);
  animation-delay: -3s;
}
.orb-c {
  width: 260px;
  height: 260px;
  top: 45%;
  right: 8%;
  background: radial-gradient(circle, rgba(56, 189, 248, 0.18), transparent 70%);
  animation-delay: -6s;
}
@keyframes orbFloat {
  0%,
  100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-18px);
  }
}

/* 필름 그레인 질감 */
.grain {
  position: fixed;
  inset: 0;
  z-index: 60;
  pointer-events: none;
  opacity: 0.04;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='160' height='160'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.9' numOctaves='2'/%3E%3C/filter%3E%3Crect width='160' height='160' filter='url(%23n)'/%3E%3C/svg%3E");
}

/* ── 메인 레이아웃 ───────────────────────────── */
.sky-main {
  position: relative;
  z-index: 1;
  max-width: 1140px;
  margin: 0 auto;
  padding: 80px 32px 120px;
}

.hero {
  text-align: center;
  margin-bottom: 56px;
  animation: riseIn 0.9s var(--sky-ease) both;
}

.eyebrow {
  display: inline-block;
  padding: 6px 14px;
  border-radius: 999px;
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.18em;
  text-transform: uppercase;
  color: var(--sky-accent);
  background: rgba(125, 211, 252, 0.08);
  border: 1px solid rgba(125, 211, 252, 0.18);
  margin-bottom: 22px;
}

.hero-title {
  font-size: clamp(2rem, 5vw, 2.9rem);
  font-weight: 800;
  line-height: 1.25;
  letter-spacing: -0.02em;
  margin: 0 0 16px;
}
.hero-title em {
  font-style: normal;
  background: linear-gradient(120deg, var(--sky-accent), var(--sky-accent-deep));
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
}

.hero-sub {
  font-size: 15px;
  color: var(--sky-ink-dim);
  line-height: 1.6;
  margin: 0;
}

@keyframes riseIn {
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

@media (max-width: 768px) {
  .sky-main {
    padding: 64px 16px 96px;
  }
}
</style>
