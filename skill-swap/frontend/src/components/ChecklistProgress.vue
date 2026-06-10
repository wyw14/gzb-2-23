<template>
  <div class="checklist-progress" @click="handleClick">
    <div class="progress-header">
      <div class="progress-title">
        <el-icon><List /></el-icon>
        <span>交换清单</span>
      </div>
      <div class="progress-rate" v-if="stats.total > 0">
        <span class="rate-number">{{ stats.completionRate }}%</span>
        <span class="rate-detail">({{ stats.completed }}/{{ stats.total }})</span>
      </div>
      <el-tag v-else size="small" type="info">暂无清单</el-tag>
    </div>

    <el-progress
      v-if="stats.total > 0"
      :percentage="stats.completionRate"
      :stroke-width="6"
      :show-text="false"
      :color="progressColor"
    />

    <div class="progress-types" v-if="stats.total > 0">
      <div class="type-item" v-for="type in typeStats" :key="type.key">
        <el-icon :color="type.color"><component :is="type.icon" /></el-icon>
        <span class="type-label">{{ type.label }}</span>
        <span class="type-count">{{ type.completed }}/{{ type.total }}</span>
      </div>
    </div>

    <div class="progress-hint" v-if="showHint && stats.total > 0 && stats.completionRate < 100">
      <el-icon><Warning /></el-icon>
      <span>还有 {{ stats.total - stats.completed }} 项未完成</span>
    </div>
  </div>
</template>

<script setup>
import { computed } from 'vue'
import { List, Warning, Reading, Notebook, Upload } from '@element-plus/icons-vue'

const props = defineProps({
  stats: {
    type: Object,
    default: () => ({ total: 0, completed: 0, completionRate: 0 })
  },
  detailStats: {
    type: Object,
    default: () => ({ material: { total: 0, completed: 0 }, practice: { total: 0, completed: 0 }, homework: { total: 0, completed: 0 } })
  },
  clickable: {
    type: Boolean,
    default: false
  },
  showHint: {
    type: Boolean,
    default: false
  }
})

const emit = defineEmits(['click'])

const progressColor = computed(() => {
  if (props.stats.completionRate === 100) return '#52c41a'
  if (props.stats.completionRate >= 60) return '#1890ff'
  if (props.stats.completionRate >= 30) return '#fa8c16'
  return '#ff4d4f'
})

const typeStats = computed(() => [
  {
    key: 'material',
    label: '资料',
    icon: Reading,
    color: '#1890ff',
    total: props.detailStats.material?.total || 0,
    completed: props.detailStats.material?.completed || 0
  },
  {
    key: 'practice',
    label: '练习',
    icon: Notebook,
    color: '#52c41a',
    total: props.detailStats.practice?.total || 0,
    completed: props.detailStats.practice?.completed || 0
  },
  {
    key: 'homework',
    label: '作业',
    icon: Upload,
    color: '#722ed1',
    total: props.detailStats.homework?.total || 0,
    completed: props.detailStats.homework?.completed || 0
  }
])

function handleClick() {
  if (props.clickable) {
    emit('click')
  }
}
</script>

<style scoped>
.checklist-progress {
  background: linear-gradient(135deg, #f9f9ff 0%, #f5f7ff 100%);
  border-radius: 10px;
  padding: 12px 14px;
  border: 1px solid #e8ebff;
  cursor: default;
  transition: all 0.2s;
}

.checklist-progress:hover {
  border-color: #667eea;
}

.progress-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 10px;
}

.progress-title {
  display: flex;
  align-items: center;
  gap: 6px;
  font-weight: 600;
  font-size: 13px;
  color: #333;
  flex: 1;
}

.progress-rate {
  display: flex;
  align-items: baseline;
  gap: 2px;
}

.rate-number {
  font-weight: 700;
  font-size: 15px;
  color: #667eea;
}

.rate-detail {
  font-size: 11px;
  color: #999;
}

.progress-types {
  display: flex;
  gap: 12px;
  margin-top: 10px;
  padding-top: 10px;
  border-top: 1px dashed #e0e0e0;
  flex-wrap: wrap;
}

.type-item {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
}

.type-label {
  color: #666;
}

.type-count {
  color: #999;
  font-size: 11px;
}

.progress-hint {
  display: flex;
  align-items: center;
  gap: 4px;
  margin-top: 10px;
  font-size: 12px;
  color: #fa8c16;
  padding: 6px 10px;
  background: #fff7e6;
  border-radius: 6px;
}
</style>
