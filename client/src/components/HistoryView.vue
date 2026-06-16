<template>
  <div class="card">
    <div v-if="history && history.stats" class="stats-panel">
      <div class="stat-card green">
        <div class="stat-num">{{ history.stats.answeredCount }}</div>
        <div class="stat-label">✓ 已回答天数</div>
      </div>
      <div class="stat-card red">
        <div class="stat-num">{{ history.stats.missedCount }}</div>
        <div class="stat-label">✗ 断更天数</div>
      </div>
      <div class="stat-card blue">
        <div class="stat-num">
          {{ history.stats.totalDays > 0 ? Math.round(history.stats.answeredCount / history.stats.totalDays * 100) : 0 }}%
        </div>
        <div class="stat-label">📊 坚持率</div>
      </div>
    </div>

    <div v-if="history && history.tags && history.tags.length > 0" class="tags-panel">
      <div class="tags-panel-header">
        <h4>🏷️ 标签统计</h4>
        <span
          v-if="activeFilterTag"
          class="filter-active-hint"
        >
          正在过滤：<strong>{{ activeFilterTag }}</strong>
          <button class="clear-filter-btn" @click="clearFilter">× 清除</button>
        </span>
      </div>
      <div class="tags-cloud">
        <button
          v-for="tag in history.tags"
          :key="tag.name"
          class="tag-filter-btn"
          :class="{ active: activeFilterTag === tag.name }"
          @click="toggleFilter(tag.name)"
        >
          <span class="tag-name">{{ tag.name }}</span>
          <span class="tag-count-badge">{{ tag.count }}</span>
        </button>
      </div>
    </div>

    <div class="calendar-header">
      <button class="nav-btn" @click="$emit('prev-month')">← 上月</button>
      <div class="calendar-title">
        {{ history?.year }}年 {{ history?.month }}月
      </div>
      <button class="nav-btn" @click="$emit('next-month')">下月 →</button>
    </div>

    <div class="calendar-grid">
      <div v-for="w in weekdays" :key="w" class="calendar-weekday">{{ w }}</div>
      <div
        v-for="(day, idx) in fullCalendar"
        :key="idx"
        class="calendar-day"
        :class="getDayClass(day)"
        @click="selectDay(day)"
      >
        <span v-if="day" class="day-num">{{ day.day }}</span>
        <span v-if="day && day.tags && day.tags.length > 0" class="day-tag-indicator"></span>
      </div>
    </div>

    <div class="legend">
      <div class="legend-item">
        <div class="legend-box answered"></div>
        <span>已回答</span>
      </div>
      <div class="legend-item">
        <div class="legend-box missed"></div>
        <span>断更（应回答未答）</span>
      </div>
      <div class="legend-item">
        <div class="legend-box today"></div>
        <span>今天</span>
      </div>
      <div class="legend-item">
        <div class="legend-box empty-day"></div>
        <span>无记录</span>
      </div>
      <div class="legend-item">
        <div class="legend-box tagged"></div>
        <span>有标签</span>
      </div>
    </div>

    <div v-if="selectedDay" class="day-detail">
      <h4>📌 {{ formatFullDate(selectedDay.date) }}</h4>
      <div v-if="selectedDay.hasQuestion">
        <p class="q-text">❓ {{ selectedDay.question }}</p>
        <div v-if="selectedDay.answered && selectedDay.answer">
          <p class="a-text">{{ selectedDay.answer }}</p>
          <div v-if="selectedDay.tags && selectedDay.tags.length > 0" class="detail-tags">
            <span class="detail-tags-label">标签：</span>
            <span v-for="(tag, idx) in selectedDay.tags" :key="idx" class="tag-chip readonly">
              {{ tag }}
            </span>
          </div>
        </div>
        <div v-else class="empty-note">这一天没有回答</div>
      </div>
      <div v-else class="empty-note">这一天还没有分配问题</div>
    </div>

    <div v-if="loading" class="empty-note">加载中...</div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
  history: Object,
  loading: Boolean
})

defineEmits(['prev-month', 'next-month'])

const weekdays = ['日', '一', '二', '三', '四', '五', '六']
const selectedDay = ref(null)
const activeFilterTag = ref(null)

const filteredCalendar = computed(() => {
  if (!props.history?.calendar) return []
  if (!activeFilterTag.value) return props.history.calendar
  return props.history.calendar.map(day => {
    if (!day) return day
    const hasTag = day.tags && day.tags.includes(activeFilterTag.value)
    return {
      ...day,
      _filteredOut: !hasTag
    }
  })
})

const fullCalendar = computed(() => {
  const cal = [...filteredCalendar.value]
  const firstDay = cal.find(d => d)
  if (!firstDay) return cal
  const d = new Date(firstDay.date)
  const firstWeekday = d.getDay()
  for (let i = 0; i < firstWeekday; i++) {
    cal.unshift(null)
  }
  return cal
})

function getTodayStr() {
  const t = new Date()
  const y = t.getFullYear()
  const m = String(t.getMonth() + 1).padStart(2, '0')
  const d = String(t.getDate()).padStart(2, '0')
  return `${y}-${m}-${d}`
}

function isPast(dateStr) {
  return dateStr < getTodayStr()
}

function getDayClass(day) {
  if (!day) return 'empty'
  const classes = []
  const todayStr = getTodayStr()
  if (day._filteredOut) {
    classes.push('filtered-out')
  }
  if (day.date === todayStr) {
    classes.push('today')
  }
  if (day.answered) {
    classes.push('answered')
  } else if (day.hasQuestion && isPast(day.date)) {
    classes.push('missed')
  } else if (day.hasQuestion) {
    classes.push('has-question')
  }
  return classes.join(' ')
}

function selectDay(day) {
  if (day && day.hasQuestion && !day._filteredOut) {
    selectedDay.value = day
  } else {
    selectedDay.value = null
  }
}

function toggleFilter(tagName) {
  if (activeFilterTag.value === tagName) {
    activeFilterTag.value = null
  } else {
    activeFilterTag.value = tagName
  }
  selectedDay.value = null
}

function clearFilter() {
  activeFilterTag.value = null
}

function formatFullDate(dateStr) {
  if (!dateStr) return ''
  const d = new Date(dateStr)
  const weekdays = ['周日', '周一', '周二', '周三', '周四', '周五', '周六']
  return `${d.getFullYear()}年${d.getMonth() + 1}月${d.getDate()}日 ${weekdays[d.getDay()]}`
}

watch(() => props.history, () => {
  selectedDay.value = null
})
</script>
