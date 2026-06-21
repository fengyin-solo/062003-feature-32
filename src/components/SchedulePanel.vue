<template>
  <div class="schedule-panel card">
    <h3>📅 今日日程安排</h3>
    <p class="hint">为每位练习生选择活动，同活动一起训练可提升默契</p>

    <div v-if="illTrainees.length > 0" class="substitute-section">
      <div class="section-header">
        <span class="section-title">🤒 休养中的练习生</span>
        <span class="section-count">{{ illTrainees.length }} 人</span>
      </div>
      <div class="ill-list">
        <div v-for="trainee in illTrainees" :key="trainee.id" class="ill-item">
          <div class="ill-info">
            <span class="ill-name">{{ trainee.name }}</span>
            <span class="ill-type">{{ trainee.illnessType || '休养中' }}</span>
            <span class="ill-days">剩余 {{ trainee.illnessDays }} 天</span>
          </div>
          <div class="substitute-suggestion">
            <span class="suggest-label">💡 建议替补：</span>
            <span class="suggest-names">
              {{ getSubstituteSuggestion(trainee).map((t) => t.name).join('、') || '暂无合适替补' }}
            </span>
          </div>
        </div>
      </div>
      <div class="impact-tip">
        ⚠️ 休养期间无法参与训练，建议为关键活动安排替补，避免进度落后
      </div>
    </div>

    <div class="schedule-list">
      <div
        v-for="trainee in schedulable"
        :key="trainee.id"
        class="schedule-row"
        :class="{ ill: trainee.illnessDays > 0 }"
      >
        <span class="name">{{ trainee.name }}</span>
        <span v-if="trainee.illnessDays > 0" class="ill-tag">🤒 休养中</span>
        <div v-else class="activity-btns">
          <button
            v-for="(act, key) in activities"
            :key="key"
            class="act-btn"
            :class="{ active: schedule[trainee.id] === key }"
            :title="`${act.label} ¥${act.moneyCost}`"
            @click="$emit('set', trainee.id, key)"
          >
            {{ act.icon }}
          </button>
        </div>
        <span v-if="schedule[trainee.id]" class="chosen">
          {{ activities[schedule[trainee.id]]?.label }}
        </span>
      </div>
    </div>

    <div class="legend">
      <span v-for="(act, key) in activities" :key="key" class="legend-item">
        {{ act.icon }} {{ act.label }}
      </span>
    </div>

    <div class="actions">
      <button class="btn ghost" @click="$emit('clear')">清空</button>
      <button class="btn primary" :disabled="!canEnd" @click="$emit('end-day')">
        结束今日 →
      </button>
    </div>
    <p v-if="!canEnd" class="warn">请为所有可安排的练习生选择日程</p>
  </div>
</template>

<script setup>
import { computed } from 'vue'
import { GAME_CONFIG } from '../config/gameConfig'

const props = defineProps({
  trainees: Array,
  schedule: Object,
  canEnd: Boolean,
})

defineEmits(['set', 'clear', 'end-day'])

const activities = GAME_CONFIG.activities
const statLabels = GAME_CONFIG.statLabels

const schedulable = computed(() =>
  props.trainees.filter((t) => t.status !== 'left')
)

const illTrainees = computed(() =>
  schedulable.value.filter((t) => t.illnessDays > 0)
)

const availableTrainees = computed(() =>
  schedulable.value.filter((t) => t.illnessDays === 0)
)

function getSubstituteSuggestion(illTrainee) {
  const typeEffects = GAME_CONFIG.events.types.illness.typeEffects?.[illTrainee.illnessType]
  const keyStats = typeEffects?.affectedStats || findKeyStats(illTrainee)

  const candidates = availableTrainees.value
    .filter((t) => t.id !== illTrainee.id)
    .map((t) => {
      let score = 0
      for (const stat of keyStats) {
        score += t.stats[stat] || 0
      }
      score -= t.fatigue * 0.3
      score -= t.stress * 0.2
      return { trainee: t, score }
    })
    .sort((a, b) => b.score - a.score)
    .slice(0, 2)
    .map((item) => item.trainee)

  return candidates
}

function findKeyStats(trainee) {
  const statEntries = Object.entries(trainee.stats)
  statEntries.sort((a, b) => b[1] - a[1])
  return statEntries.slice(0, 2).map(([stat]) => stat)
}
</script>

<style scoped>
.schedule-panel h3 { margin-bottom: 0.25rem; }

.hint {
  font-size: 0.85rem;
  color: var(--text-muted);
  margin-bottom: 1rem;
}

.schedule-list {
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
  margin-bottom: 1rem;
}

.schedule-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.name {
  width: 72px;
  font-weight: 600;
  font-size: 0.9rem;
}

.activity-btns {
  display: flex;
  gap: 0.35rem;
}

.act-btn {
  width: 36px;
  height: 36px;
  border: 1px solid var(--border);
  border-radius: 8px;
  background: var(--bg-secondary);
  cursor: pointer;
  font-size: 1rem;
  transition: all 0.15s;
}

.act-btn:hover { border-color: var(--accent); }
.act-btn.active {
  background: var(--accent-soft);
  border-color: var(--accent);
  transform: scale(1.1);
}

.chosen {
  font-size: 0.8rem;
  color: var(--accent);
}

.ill-tag {
  font-size: 0.8rem;
  color: var(--warning);
}

.legend {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem 1rem;
  font-size: 0.75rem;
  color: var(--text-muted);
  margin-bottom: 1rem;
}

.actions {
  display: flex;
  gap: 0.75rem;
  justify-content: flex-end;
}

.warn {
  text-align: right;
  font-size: 0.8rem;
  color: var(--warning);
  margin-top: 0.5rem;
}

.substitute-section {
  background: var(--warning-soft);
  border: 1px solid var(--warning);
  border-radius: 8px;
  padding: 0.75rem;
  margin-bottom: 1rem;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
}

.section-title {
  font-weight: 600;
  font-size: 0.9rem;
}

.section-count {
  font-size: 0.75rem;
  color: var(--text-muted);
  background: var(--bg-card);
  padding: 0.15rem 0.5rem;
  border-radius: 999px;
}

.ill-list {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.ill-item {
  background: var(--bg-card);
  border-radius: 6px;
  padding: 0.5rem 0.75rem;
}

.ill-info {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.25rem;
  flex-wrap: wrap;
}

.ill-name {
  font-weight: 600;
  font-size: 0.85rem;
}

.ill-type {
  font-size: 0.7rem;
  color: var(--warning-dark);
  background: var(--warning-soft);
  padding: 0.1rem 0.4rem;
  border-radius: 4px;
}

.ill-days {
  font-size: 0.75rem;
  color: var(--text-muted);
}

.substitute-suggestion {
  font-size: 0.75rem;
  color: var(--text-secondary);
}

.suggest-label {
  color: var(--text-muted);
}

.suggest-names {
  color: var(--accent);
  font-weight: 500;
}

.impact-tip {
  margin-top: 0.5rem;
  font-size: 0.75rem;
  color: var(--warning-dark);
  padding-top: 0.5rem;
  border-top: 1px dashed var(--warning);
}

.schedule-row.ill {
  opacity: 0.7;
}
</style>
