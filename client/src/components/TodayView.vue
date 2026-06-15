<template>
  <div class="card">
    <span class="date-label">📅 {{ formatDate(data.date) }}</span>

    <div v-if="data.answered" class="status-badge status-answered">
      ✓ 今日已回答
    </div>
    <div v-else class="status-badge status-unanswered">
      ⏰ 等待你的回答
    </div>

    <div class="question-text">
      {{ data.question.question }}
    </div>

    <label class="answer-label">✍️ 我的回答：</label>

    <textarea
      v-model="localAnswer"
      class="answer-textarea"
      placeholder="在这里写下你的思考和答案..."
      :disabled="submitting"
    ></textarea>

    <div class="tags-section">
      <label class="answer-label">🏷️ 标签（按回车添加，点击移除）：</label>
      <div class="tags-input-wrapper">
        <div class="tags-list">
          <span
            v-for="(tag, idx) in localTags"
            :key="idx"
            class="tag-chip"
          >
            {{ tag }}
            <button
              type="button"
              class="tag-remove"
              @click="removeTag(idx)"
              :disabled="submitting"
            >
              ×
            </button>
          </span>
        </div>
        <input
          type="text"
          v-model="tagInput"
          class="tag-input"
          placeholder="输入标签后按回车..."
          @keydown.enter.prevent="addTag"
          :disabled="submitting"
        />
      </div>
    </div>

    <button
      class="submit-btn"
      :disabled="submitting || !localAnswer.trim()"
      @click="handleSubmit"
    >
      {{ submitting ? '保存中...' : '💾 保存回答' }}
    </button>

    <div v-if="data.answered && data.answer" class="displayed-answer">
      <h4>📝 已保存的回答：</h4>
      <p>{{ data.answer }}</p>
      <div v-if="data.tags && data.tags.length > 0" class="saved-tags">
        <span class="saved-tags-label">标签：</span>
        <span v-for="(tag, idx) in data.tags" :key="idx" class="tag-chip readonly">
          {{ tag }}
        </span>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch } from 'vue'

const props = defineProps({
  todayData: Object,
  submitting: Boolean
})

const emit = defineEmits(['submit'])

const data = ref(props.todayData)
const localAnswer = ref('')
const localTags = ref([])
const tagInput = ref('')

watch(() => props.todayData, (newVal) => {
  if (newVal) {
    data.value = newVal
    localAnswer.value = newVal.answer || ''
    localTags.value = newVal.tags ? [...newVal.tags] : []
  }
}, { immediate: true })

function formatDate(dateStr) {
  if (!dateStr) return ''
  const d = new Date(dateStr)
  const weekdays = ['周日', '周一', '周二', '周三', '周四', '周五', '周六']
  return `${d.getFullYear()}年${d.getMonth() + 1}月${d.getDate()}日 ${weekdays[d.getDay()]}`
}

function addTag() {
  const tag = tagInput.value.trim()
  if (tag && !localTags.value.includes(tag)) {
    localTags.value.push(tag)
  }
  tagInput.value = ''
}

function removeTag(idx) {
  localTags.value.splice(idx, 1)
}

function handleSubmit() {
  emit('submit', {
    answer: localAnswer.value,
    tags: localTags.value
  })
}
</script>
