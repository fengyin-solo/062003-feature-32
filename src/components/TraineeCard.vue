<template>
  <div class="trainee-card card" :class="statusClass">
    <div class="card-top">
      <h4>{{ trainee.name }}</h4>
      <span class="badge" :class="trainee.status">{{ statusLabel }}</span>
    </div>

    <div v-if="trainee.illnessDays > 0" class="illness-panel">
      <div class="illness-header">
        <span class="illness-icon">🤒</span>
        <div class="illness-info">
          <div class="illness-type">{{ illnessTypeLabel }}</div>
          <div class="illness-days">剩余 {{ trainee.illnessDays }} 天 · 第 {{ passedDays + 1 }}/{{ trainee.totalIllnessDays }} 天</div>
        </div>
      </div>
      <div class="recovery-bar">
        <div class="recovery-fill" :style="{ width: recoveryProgress + '%' }"></div>
      </div>
      <div class="illness-effects">
        <div class="effect-item" v-for="effect in illnessEffects" :key="effect.label">
          <span class="effect-icon">{{ effect.icon }}</span>
          <span class="effect-text">{{ effect.label }}</span>
        </div>
      </div>
    </div>

    <div class="bars">
      <div class="bar-row">
        <span>疲劳</span>
        <div class="bar"><div class="fill fatigue" :style="{ width: trainee.fatigue + '%' }"></div></div>
        <span>{{ trainee.fatigue }}</span>
      </div>
      <div class="bar-row">
        <span>压力</span>
        <div class="bar"><div class="fill stress" :style="{ width: trainee.stress + '%' }"></div></div>
        <span>{{ trainee.stress }}</span>
      </div>
    </div>

    <div class="stats-grid">
      <div v-for="key in statKeys" :key="key" class="stat-cell" :class="{ 'at-risk': isStatAtRisk(key) }">
        <span class="stat-label">{{ statLabels[key] }}</span>
        <span class="stat-val">{{ trainee.stats[key] }}</span>
      </div>
    </div>

    <div v-if="score !== null" class="score">
      综合评分 <strong>{{ score }}</strong>
      <span v-if="trainee.status === 'trainee'" class="debut-hint">
        {{ score >= debutThreshold ? '✓ 可出道' : `需 ${debutThreshold}` }}
      </span>
    </div>

    <div v-if="trainee.illnessDays > 0" class="illness-tip">
      💡 休养期间每天恢复疲劳与压力，但能力可能因缺乏练习而下降
    </div>
    <div v-if="trainee.fans > 0" class="fans">个人粉丝 {{ trainee.fans.toLocaleString() }}</div>
  </div>
</template>

<script setup>
import { computed } from 'vue'
import { GAME_CONFIG } from '../config/gameConfig'

const props = defineProps({
  trainee: Object,
  score: { type: Number, default: null },
})

const statKeys = GAME_CONFIG.stats
const statLabels = GAME_CONFIG.statLabels
const debutThreshold = GAME_CONFIG.rating.debutScoreThreshold

const statusLabel = computed(() => {
  const map = { trainee: '练习生', debuted: '已出道', left: '已离开' }
  return map[props.trainee.status] || props.trainee.status
})

const statusClass = computed(() => ({
  debuted: props.trainee.status === 'debuted',
  left: props.trainee.status === 'left',
  ill: props.trainee.illnessDays > 0,
}))

const passedDays = computed(() => {
  if (!props.trainee.totalIllnessDays) return 0
  return props.trainee.totalIllnessDays - props.trainee.illnessDays
})

const recoveryProgress = computed(() => {
  if (!props.trainee.totalIllnessDays) return 0
  return Math.round((passedDays.value / props.trainee.totalIllnessDays) * 100)
})

const illnessTypeLabel = computed(() => {
  return props.trainee.illnessType || '休养中'
})

const illnessEffects = computed(() => {
  const effects = []
  const typeEffects = GAME_CONFIG.events.types.illness.typeEffects?.[props.trainee.illnessType]

  if (typeEffects?.affectedStats) {
    const affectedLabels = typeEffects.affectedStats.map((s) => statLabels[s]).join('、')
    effects.push({ icon: '📉', label: `${affectedLabels}易退步` })
  }

  effects.push({ icon: '😴', label: '无法参加训练' })
  effects.push({ icon: '💰', label: '不产生训练费用' })

  return effects
})

function isStatAtRisk(statKey) {
  if (props.trainee.illnessDays <= 0) return false
  const typeEffects = GAME_CONFIG.events.types.illness.typeEffects?.[props.trainee.illnessType]
  return typeEffects?.affectedStats?.includes(statKey) || false
}
</script>

<style scoped>
.trainee-card {
  padding: 1rem;
  transition: border-color 0.2s;
}

.trainee-card.debuted { border-color: var(--accent); }
.trainee-card.left { opacity: 0.5; }
.trainee-card.ill { border-color: var(--warning); }

.card-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
}

.card-top h4 { font-size: 1rem; }

.badge {
  font-size: 0.7rem;
  padding: 0.15rem 0.5rem;
  border-radius: 999px;
  background: var(--bg-secondary);
}

.badge.debuted { background: var(--accent-soft); color: var(--accent); }
.badge.left { background: var(--danger-soft); color: var(--danger); }

.bars { margin-bottom: 0.75rem; }

.bar-row {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.75rem;
  margin-bottom: 0.25rem;
}

.bar-row span:first-child { width: 28px; color: var(--text-muted); }
.bar-row span:last-child { width: 24px; text-align: right; }

.bar {
  flex: 1;
  height: 6px;
  background: var(--bg-secondary);
  border-radius: 3px;
  overflow: hidden;
}

.fill { height: 100%; border-radius: 3px; transition: width 0.3s; }
.fill.fatigue { background: var(--warning); }
.fill.stress { background: var(--danger); }

.stats-grid {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 0.25rem;
  text-align: center;
}

.stat-cell {
  background: var(--bg-secondary);
  border-radius: 6px;
  padding: 0.3rem 0.1rem;
}

.stat-label { display: block; font-size: 0.65rem; color: var(--text-muted); }
.stat-val { font-weight: 700; font-size: 0.85rem; }

.score {
  margin-top: 0.5rem;
  font-size: 0.85rem;
  color: var(--text-secondary);
}

.debut-hint {
  margin-left: 0.5rem;
  font-size: 0.75rem;
  color: var(--accent);
}

.illness-panel {
  background: var(--warning-soft);
  border: 1px solid var(--warning);
  border-radius: 8px;
  padding: 0.75rem;
  margin-bottom: 0.75rem;
}

.illness-header {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 0.5rem;
}

.illness-icon {
  font-size: 1.5rem;
}

.illness-info {
  flex: 1;
}

.illness-type {
  font-weight: 600;
  font-size: 0.9rem;
  color: var(--warning-dark);
}

.illness-days {
  font-size: 0.75rem;
  color: var(--text-muted);
  margin-top: 0.1rem;
}

.recovery-bar {
  height: 6px;
  background: var(--bg-secondary);
  border-radius: 3px;
  overflow: hidden;
  margin-bottom: 0.5rem;
}

.recovery-fill {
  height: 100%;
  background: linear-gradient(90deg, var(--warning), var(--success));
  border-radius: 3px;
  transition: width 0.3s;
}

.illness-effects {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.effect-item {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  font-size: 0.7rem;
  color: var(--text-secondary);
  background: var(--bg-card);
  padding: 0.2rem 0.5rem;
  border-radius: 4px;
}

.effect-icon {
  font-size: 0.8rem;
}

.stat-cell.at-risk {
  background: var(--warning-soft);
  border: 1px dashed var(--warning);
}

.stat-cell.at-risk .stat-label {
  color: var(--warning-dark);
}

.illness-tip {
  margin-top: 0.5rem;
  font-size: 0.75rem;
  color: var(--text-muted);
  padding: 0.4rem 0.6rem;
  background: var(--bg-secondary);
  border-radius: 6px;
}

.fans {
  margin-top: 0.35rem;
  font-size: 0.8rem;
  color: var(--accent);
}
</style>
