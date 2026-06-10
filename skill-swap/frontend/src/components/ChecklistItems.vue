<template>
  <div class="checklist-items">
    <div
      v-for="item in sortedItems"
      :key="item.id"
      class="checklist-item"
      :class="{ completed: item.completed }"
    >
      <div class="item-checkbox" @click="handleToggle(item)">
        <el-checkbox
          :model-value="item.completed"
          :disabled="!canToggle(item)"
          @click.stop
        />
      </div>

      <div class="item-content">
        <div class="item-header">
          <span class="item-title" :class="{ 'line-through': item.completed }">
            {{ item.title }}
          </span>
          <el-tag
            size="small"
            :type="getTypeTagType(item.type)"
            effect="light"
          >
            {{ getTypeLabel(item.type) }}
          </el-tag>
        </div>

        <div class="item-meta" v-if="item.description || item.url">
          <p class="item-description" v-if="item.description">
            {{ item.description }}
          </p>
          <a
            v-if="item.url"
            class="item-url"
            :href="item.url"
            target="_blank"
            rel="noopener noreferrer"
          >
            <el-icon><Link /></el-icon>
            {{ item.url }}
          </a>
        </div>

        <div class="item-footer">
          <div class="item-owner">
            <el-avatar :size="20" :src="getUserAvatar(item.ownerId)" />
            <span class="owner-name">
              负责人: {{ getUserName(item.ownerId) }}
              <span v-if="item.ownerId === myId" class="me-tag">(我)</span>
            </span>
          </div>
          <div class="item-status" v-if="item.completed">
            <el-icon color="#52c41a"><CircleCheck /></el-icon>
            <span class="completed-text">
              {{ getUserName(item.completedBy) }} 于 {{ formatTime(item.completedAt) }} 完成
            </span>
          </div>
          <div class="item-actions" v-if="canEdit(item)">
            <el-button
              type="primary"
              link
              size="small"
              @click="$emit('edit', item)"
            >
              <el-icon><Edit /></el-icon>编辑
            </el-button>
            <el-button
              type="danger"
              link
              size="small"
              @click="$emit('delete', item)"
            >
              <el-icon><Delete /></el-icon>删除
            </el-button>
          </div>
        </div>
      </div>
    </div>
    <el-empty v-if="items.length === 0" description="暂无项目" :image-size="60" />
  </div>
</template>

<script setup>
import { computed } from 'vue'
import dayjs from 'dayjs'
import { Link, CircleCheck, Edit, Delete } from '@element-plus/icons-vue'

const props = defineProps({
  items: {
    type: Array,
    default: () => []
  },
  myId: {
    type: String,
    required: true
  },
  users: {
    type: Object,
    default: () => ({})
  }
})

const emit = defineEmits(['toggle', 'edit', 'delete'])

const sortedItems = computed(() => {
  return [...props.items].sort((a, b) => {
    if (a.completed !== b.completed) return a.completed ? 1 : -1
    return new Date(a.createdAt) - new Date(b.createdAt)
  })
})

function getTypeLabel(type) {
  const labels = {
    material: '课前资料',
    practice: '练习任务',
    homework: '作业提交'
  }
  return labels[type] || type
}

function getTypeTagType(type) {
  const types = {
    material: 'primary',
    practice: 'success',
    homework: 'warning'
  }
  return types[type] || 'info'
}

function getUserName(id) {
  return props.users[id]?.username || '未知用户'
}

function getUserAvatar(id) {
  return props.users[id]?.avatar || ''
}

function formatTime(time) {
  if (!time) return ''
  return dayjs(time).format('MM-DD HH:mm')
}

function canToggle(item) {
  return item.ownerId === props.myId
}

function canEdit(item) {
  return item.createdBy === props.myId
}

function handleToggle(item) {
  if (canToggle(item)) {
    emit('toggle', item)
  }
}
</script>

<style scoped>
.checklist-items {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.checklist-item {
  display: flex;
  gap: 12px;
  padding: 16px;
  background: #fafafa;
  border-radius: 10px;
  border: 1px solid #f0f0f0;
  transition: all 0.2s;
}

.checklist-item:hover {
  border-color: #667eea;
  background: #f9f9ff;
}

.checklist-item.completed {
  background: #f6ffed;
  border-color: #b7eb8f;
}

.item-checkbox {
  flex-shrink: 0;
  padding-top: 2px;
  cursor: pointer;
}

.item-content {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.item-header {
  display: flex;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}

.item-title {
  flex: 1;
  font-weight: 600;
  color: #333;
  font-size: 15px;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
}

.line-through {
  text-decoration: line-through;
  color: #999;
}

.item-meta {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.item-description {
  margin: 0;
  font-size: 13px;
  color: #666;
  line-height: 1.6;
}

.item-url {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 13px;
  color: #667eea;
  text-decoration: none;
  max-width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.item-url:hover {
  text-decoration: underline;
}

.item-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 8px;
  padding-top: 4px;
  border-top: 1px dashed #e8e8e8;
}

.item-owner {
  display: flex;
  align-items: center;
  gap: 6px;
}

.owner-name {
  font-size: 12px;
  color: #999;
}

.me-tag {
  color: #667eea;
  font-weight: 500;
}

.item-status {
  display: flex;
  align-items: center;
  gap: 4px;
}

.completed-text {
  font-size: 12px;
  color: #52c41a;
}

.item-actions {
  display: flex;
  gap: 4px;
}
</style>
