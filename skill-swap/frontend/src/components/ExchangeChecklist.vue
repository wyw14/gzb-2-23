<template>
  <div class="exchange-checklist">
    <div class="checklist-header">
      <div class="header-left">
        <h3 class="checklist-title">
          <el-icon><List /></el-icon>
          材料与作业清单
        </h3>
        <div class="completion-badge" v-if="stats.total > 0">
          <el-progress
            :percentage="stats.completionRate"
            :stroke-width="8"
            :color="progressColor"
          />
          <span class="completion-text">{{ stats.completed }}/{{ stats.total }} 已完成</span>
        </div>
      </div>
      <el-button type="primary" size="small" @click="showAddDialog = true">
        <el-icon><Plus /></el-icon>添加项目
      </el-button>
    </div>

    <div class="checklist-tabs" v-if="Object.keys(groupedChecklists).length > 0">
      <div
        v-for="tab in tabs"
        :key="tab.type"
        class="tab-item"
        :class="{ active: activeType === tab.type }"
        @click="activeType = tab.type"
      >
        <el-icon><component :is="tab.icon" /></el-icon>
        <span>{{ tab.label }}</span>
        <span class="tab-count">({{ getCountByType(tab.type) }})</span>
      </div>
    </div>

    <div class="checklist-content">
      <template v-if="activeType === 'all'">
        <div v-for="tab in tabs.filter(t => t.type !== 'all')" :key="tab.type" class="type-section">
          <div class="type-header" v-if="groupedChecklists[tab.type]?.length > 0">
            <el-icon :color="tab.color"><component :is="tab.icon" /></el-icon>
            <span class="type-name">{{ tab.label }}</span>
            <span class="type-count">
              {{ getCompletedCountByType(tab.type) }}/{{ groupedChecklists[tab.type]?.length || 0 }}
            </span>
          </div>
          <ChecklistItems
            :items="groupedChecklists[tab.type] || []"
            :my-id="myId"
            :users="users"
            @toggle="handleToggle"
            @edit="handleEdit"
            @delete="handleDelete"
          />
        </div>
      </template>
      <template v-else>
        <ChecklistItems
          :items="groupedChecklists[activeType] || []"
          :my-id="myId"
          :users="users"
          @toggle="handleToggle"
          @edit="handleEdit"
          @delete="handleDelete"
        />
      </template>
    </div>

    <el-empty v-if="checklists.length === 0" description="暂无清单项目，点击右上角添加" />

    <el-dialog
      v-model="showAddDialog"
      :title="editingItem ? '编辑清单项目' : '添加清单项目'"
      width="520px"
      @closed="resetForm"
    >
      <el-form :model="checklistForm" label-position="top">
        <el-form-item label="项目类型" required>
          <el-radio-group v-model="checklistForm.type">
            <el-radio value="material">
              <el-icon><Reading /></el-icon>课前资料
            </el-radio>
            <el-radio value="practice">
              <el-icon><EditPen /></el-icon>练习任务
            </el-radio>
            <el-radio value="homework">
              <el-icon><FolderOpened /></el-icon>作业提交
            </el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="项目标题" required>
          <el-input v-model="checklistForm.title" placeholder="例如：Python基础教程文档" maxlength="100" show-word-limit />
        </el-form-item>
        <el-form-item label="详细描述">
          <el-input
            v-model="checklistForm.description"
            type="textarea"
            :rows="3"
            placeholder="补充说明，如完成要求、注意事项等..."
            maxlength="500"
            show-word-limit
          />
        </el-form-item>
        <el-form-item label="相关链接" v-if="checklistForm.type !== 'practice'">
          <el-input
            v-model="checklistForm.url"
            :placeholder="checklistForm.type === 'material' ? '输入资料链接，如文档、网盘地址等' : '输入作业提交链接，如GitHub、作业平台地址等'"
          />
        </el-form-item>
        <el-form-item label="负责人">
          <el-select v-model="checklistForm.ownerId" placeholder="选择负责人，只有负责人能标记完成" style="width: 100%">
            <el-option :label="`我 (${getUserName(myId)})`" :value="myId" />
            <el-option :label="`${getUserName(partnerId)} (对方)`" :value="partnerId" />
          </el-select>
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="showAddDialog = false">取消</el-button>
        <el-button type="primary" @click="submitChecklist" :loading="submitting">
          {{ editingItem ? '保存修改' : '确认添加' }}
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { exchangeAPI } from '../api'
import { useUserStore } from '../stores/user'
import { ElMessage, ElMessageBox } from 'element-plus'
import {
  List, Plus, Reading, EditPen, FolderOpened,
  Collection, Notebook, Upload
} from '@element-plus/icons-vue'
import ChecklistItems from './ChecklistItems.vue'

const props = defineProps({
  exchangeId: {
    type: String,
    required: true
  },
  initiatorId: {
    type: String,
    required: true
  },
  partnerId: {
    type: String,
    required: true
  },
  users: {
    type: Object,
    default: () => ({})
  }
})

const emit = defineEmits(['update:stats'])

const userStore = useUserStore()
const myId = userStore.user?.id

const checklists = ref([])
const activeType = ref('all')
const showAddDialog = ref(false)
const editingItem = ref(null)
const submitting = ref(false)
const checklistForm = ref({
  type: 'material',
  title: '',
  description: '',
  url: '',
  ownerId: myId
})

const tabs = [
  { type: 'all', label: '全部', icon: Collection, color: '#667eea' },
  { type: 'material', label: '课前资料', icon: Reading, color: '#1890ff' },
  { type: 'practice', label: '练习任务', icon: Notebook, color: '#52c41a' },
  { type: 'homework', label: '作业提交', icon: Upload, color: '#722ed1' }
]

const stats = computed(() => {
  const total = checklists.value.length
  const completed = checklists.value.filter(c => c.completed).length
  return {
    total,
    completed,
    completionRate: total > 0 ? Math.round((completed / total) * 100) : 0
  }
})

watch(stats, (newStats) => {
  emit('update:stats', newStats)
}, { immediate: true, deep: true })

const progressColor = computed(() => {
  if (stats.value.completionRate === 100) return '#52c41a'
  if (stats.value.completionRate >= 60) return '#1890ff'
  if (stats.value.completionRate >= 30) return '#fa8c16'
  return '#ff4d4f'
})

const groupedChecklists = computed(() => {
  const groups = { material: [], practice: [], homework: [] }
  checklists.value.forEach(item => {
    if (groups[item.type]) {
      groups[item.type].push(item)
    }
  })
  return groups
})

function getCountByType(type) {
  if (type === 'all') return checklists.value.length
  return groupedChecklists.value[type]?.length || 0
}

function getCompletedCountByType(type) {
  return (groupedChecklists.value[type] || []).filter(c => c.completed).length
}

function getUserName(id) {
  return props.users[id]?.username || '未知用户'
}

onMounted(async () => {
  await loadChecklists()
})

async function loadChecklists() {
  try {
    const res = await exchangeAPI.getChecklists(props.exchangeId)
    checklists.value = res.data
  } catch (e) {
    ElMessage.error('加载清单失败')
  }
}

function resetForm() {
  editingItem.value = null
  checklistForm.value = {
    type: 'material',
    title: '',
    description: '',
    url: '',
    ownerId: myId
  }
}

function handleEdit(item) {
  editingItem.value = item
  checklistForm.value = {
    type: item.type,
    title: item.title,
    description: item.description || '',
    url: item.url || '',
    ownerId: item.ownerId
  }
  showAddDialog.value = true
}

async function handleDelete(item) {
  try {
    await ElMessageBox.confirm('确定要删除这个清单项目吗？', '确认删除', {
      type: 'warning'
    })
    await exchangeAPI.deleteChecklist(item.id)
    ElMessage.success('删除成功')
    await loadChecklists()
  } catch (e) {
    if (e !== 'cancel') {
      ElMessage.error(e.message || '删除失败')
    }
  }
}

async function handleToggle(item) {
  try {
    await exchangeAPI.toggleChecklist(item.id)
    ElMessage.success(item.completed ? '已取消完成' : '已标记完成')
    await loadChecklists()
  } catch (e) {
    ElMessage.error(e.message || '操作失败')
  }
}

async function submitChecklist() {
  if (!checklistForm.value.title.trim()) {
    ElMessage.warning('请填写项目标题')
    return
  }
  if (!checklistForm.value.type) {
    ElMessage.warning('请选择项目类型')
    return
  }

  try {
    submitting.value = true
    if (editingItem.value) {
      await exchangeAPI.updateChecklist(editingItem.value.id, checklistForm.value)
      ElMessage.success('修改成功')
    } else {
      await exchangeAPI.createChecklist(props.exchangeId, checklistForm.value)
      ElMessage.success('添加成功')
    }
    showAddDialog.value = false
    resetForm()
    await loadChecklists()
  } catch (e) {
    ElMessage.error(e.message || '保存失败')
  } finally {
    submitting.value = false
  }
}
</script>

<style scoped>
.exchange-checklist {
  background: white;
  border-radius: 12px;
  padding: 20px;
  border: 1px solid #f0f0f0;
}

.checklist-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 20px;
  gap: 16px;
}

.header-left {
  flex: 1;
}

.checklist-title {
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 0 0 12px 0;
  font-size: 18px;
  color: #333;
}

.completion-badge {
  display: flex;
  align-items: center;
  gap: 16px;
  background: #f9f9ff;
  padding: 12px 16px;
  border-radius: 8px;
  max-width: 360px;
}

.completion-badge :deep(.el-progress) {
  flex: 1;
}

.completion-text {
  font-size: 13px;
  color: #666;
  white-space: nowrap;
}

.checklist-tabs {
  display: flex;
  gap: 4px;
  margin-bottom: 16px;
  padding-bottom: 12px;
  border-bottom: 1px solid #f0f0f0;
  flex-wrap: wrap;
}

.tab-item {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 16px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
  color: #666;
  transition: all 0.2s;
  user-select: none;
}

.tab-item:hover {
  background: #f5f7ff;
  color: #667eea;
}

.tab-item.active {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.tab-count {
  opacity: 0.7;
  font-size: 12px;
}

.checklist-content {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.type-section {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.type-header {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 0;
  font-weight: 600;
  color: #333;
}

.type-name {
  flex: 1;
}

.type-count {
  font-size: 12px;
  color: #999;
  font-weight: normal;
}
</style>
