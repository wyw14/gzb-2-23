<template>
  <div class="chat-page">
    <div class="chat-layout">
      <div class="sidebar">
        <div class="sidebar-header">
          <h3>消息列表</h3>
        </div>
        <div class="conversation-list">
          <div
            v-for="conv in conversations"
            :key="conv.userId"
            class="conversation-item"
            :class="{ active: currentUserId === conv.userId }"
            @click="selectConversation(conv.userId)"
          >
            <el-avatar :src="conv.avatar" :size="48" />
            <div class="conv-info">
              <div class="conv-header">
                <span class="conv-name">{{ conv.username }}</span>
                <span class="conv-time">{{ formatTime(conv.lastMessageTime) }}</span>
              </div>
              <div class="conv-message">
                <span class="last-msg">{{ conv.lastMessage }}</span>
                <el-badge v-if="conv.unreadCount > 0" :value="conv.unreadCount" class="unread-badge" />
              </div>
              <div class="conv-checklist-mini" v-if="conversationChecklistStats[conv.userId] && conversationChecklistStats[conv.userId].total > 0">
                <el-icon size="12" color="#667eea"><List /></el-icon>
                <span class="mini-text">
                  清单 {{ conversationChecklistStats[conv.userId].completed }}/{{ conversationChecklistStats[conv.userId].total }}
                </span>
                <el-progress
                  :percentage="conversationChecklistStats[conv.userId].completionRate"
                  :stroke-width="3"
                  :show-text="false"
                  :color="getProgressColor(conversationChecklistStats[conv.userId].completionRate)"
                  style="flex: 1; max-width: 80px;"
                />
              </div>
            </div>
          </div>
          <el-empty v-if="conversations.length === 0" description="暂无消息" />
        </div>
      </div>

      <div class="chat-main">
        <div v-if="currentUser" class="chat-header">
          <el-avatar :src="currentUser.avatar" :size="40" />
          <div class="header-user-info">
            <span class="chat-username">{{ currentUser.username }}</span>
            <div class="header-checklist-status" v-if="activeExchange">
              <el-icon size="12" color="#667eea"><List /></el-icon>
              <span class="status-text">
                清单完成度: {{ activeExchange.checklistStats?.completionRate || 0 }}%
                ({{ activeExchange.checklistStats?.completed || 0 }}/{{ activeExchange.checklistStats?.total || 0 }})
              </span>
              <el-button type="primary" link size="small" @click="showChecklistPanel = true">
                查看详情
              </el-button>
            </div>
          </div>
          <div class="header-actions">
            <el-button type="primary" size="small" @click="showChecklistPanel = true" v-if="activeExchange">
              <el-icon><List /></el-icon>清单
            </el-button>
            <el-button type="primary" size="small" @click="showExchangeDialog = true">
              <el-icon><Connection /></el-icon>发起交换
            </el-button>
          </div>
        </div>

        <div v-if="currentUserId" class="messages-container" ref="messagesContainer">
          <div v-for="msg in messages" :key="msg.id" class="message-row" :class="msg.senderId === myId ? 'sent' : 'received'">
            <el-avatar v-if="msg.senderId !== myId" :src="currentUser?.avatar" :size="32" />
            <div class="chat-bubble" :class="msg.senderId === myId ? 'sent' : 'received'">
              {{ msg.content }}
            </div>
          </div>
          <el-empty v-if="messages.length === 0" description="开始你们的对话吧" />
        </div>

        <div v-else class="chat-empty">
          <el-icon size="64"><ChatDotRound /></el-icon>
          <p>选择一个会话开始聊天</p>
        </div>

        <div v-if="currentUserId" class="chat-input-area">
          <el-input
            v-model="messageInput"
            placeholder="输入消息..."
            size="large"
            @keyup.enter="sendMessage"
          >
            <template #append>
              <el-button type="primary" @click="sendMessage" :loading="sending">
                <el-icon><Promotion /></el-icon>发送
              </el-button>
            </template>
          </el-input>
        </div>
      </div>

      <el-drawer
        v-model="showChecklistPanel"
        :title="checklistPanelTitle"
        direction="rtl"
        size="480px"
        @close="onChecklistPanelClose"
      >
        <div v-if="activeExchange" class="checklist-drawer-wrapper">
          <div class="drawer-exchange-info">
            <div class="exchange-info-card">
              <div class="info-users-row">
                <el-avatar :src="getMyAvatar()" :size="36" />
                <el-icon class="info-icon"><Switch /></el-icon>
                <el-avatar :src="currentUser?.avatar" :size="36" />
              </div>
              <div class="info-skills-row">
                <div class="skill-badge teach">
                  <el-icon><Upload /></el-icon>
                  <span>教授: {{ getMyTeachSkillsForExchange(activeExchange) }}</span>
                </div>
                <div class="skill-badge learn">
                  <el-icon><Download /></el-icon>
                  <span>学习: {{ getMyLearnSkillsForExchange(activeExchange) }}</span>
                </div>
              </div>
              <div class="exchange-status-row" :class="activeExchange.status">
                <el-tag :type="activeExchange.status === 'completed' ? 'success' : 'warning'" size="small">
                  {{ activeExchange.status === 'completed' ? '已完成' : `待确认 ${activeExchange.confirmedBy?.length || 0}/2` }}
                </el-tag>
                <span class="exchange-time-small">
                  {{ activeExchange.status === 'completed' ? '完成于' : '创建于' }}
                  {{ formatFullTime(activeExchange.status === 'completed' ? activeExchange.completedAt : activeExchange.createdAt) }}
                </span>
              </div>
            </div>
          </div>

          <ChecklistProgress
            :stats="activeExchange.checklistStats || { total: 0, completed: 0, completionRate: 0 }"
            :detail-stats="activeChecklistDetailStats"
            :show-hint="activeExchange.checklistStats && activeExchange.checklistStats.total > 0 && activeExchange.checklistStats.completionRate < 100 && activeExchange.status !== 'completed'"
          />

          <div class="drawer-divider">
            <span>清单项目</span>
          </div>

          <ExchangeChecklist
            v-if="activeExchange"
            :exchange-id="activeExchange.id"
            :initiator-id="activeExchange.initiatorId"
            :partner-id="activeExchange.partnerId"
            :users="chatUsers"
            @update:stats="onChecklistStatsUpdate"
          />
        </div>

        <div v-else-if="currentUser" class="drawer-empty">
          <el-icon size="64" color="#ccc"><List /></el-icon>
          <h4>暂无进行中的交换</h4>
          <p>你与 {{ currentUser.username }} 之间还没有发起技能交换</p>
          <el-button type="primary" @click="showExchangeDialog = true; showChecklistPanel = false">
            <el-icon><Connection /></el-icon>发起技能交换
          </el-button>
        </div>

        <template #footer v-if="activeExchange && activeExchange.status !== 'completed'">
          <div class="drawer-footer">
            <el-button @click="showChecklistPanel = false">关闭</el-button>
            <el-button
              type="primary"
              @click="confirmExchangeFromDrawer"
              :disabled="activeExchange.confirmedBy?.includes(myId)"
            >
              <el-icon><Check /></el-icon>
              {{ activeExchange.confirmedBy?.includes(myId) ? '已确认完成' : '确认完成交换' }}
            </el-button>
          </div>
        </template>
      </el-drawer>
    </div>

    <el-dialog v-model="showExchangeDialog" title="发起技能交换" width="500px">
      <el-form :model="exchangeForm" label-position="top">
        <el-form-item label="你想教授的技能">
          <el-select v-model="exchangeForm.teachSkill" placeholder="选择你可以教的技能" style="width: 100%">
            <el-option v-for="s in myTeachSkills" :key="s.id" :label="s.name" :value="s.name" />
          </el-select>
        </el-form-item>
        <el-form-item label="你想学习的技能">
          <el-select v-model="exchangeForm.learnSkill" placeholder="选择你想要学习的技能" style="width: 100%">
            <el-option v-for="s in otherTeachSkills" :key="s.id" :label="s.name" :value="s.name" />
          </el-select>
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="showExchangeDialog = false">取消</el-button>
        <el-button type="primary" @click="createExchange">确认发起</el-button>
      </template>
    </el-dialog>

    <el-dialog v-model="showConfirmFromDrawer" title="确认完成交换" width="460px">
      <div v-if="activeExchange" class="confirm-dialog-inner">
        <div class="confirm-warning-box" v-if="activeExchange.checklistStats && activeExchange.checklistStats.completionRate < 100 && activeExchange.checklistStats.total > 0">
          <el-icon size="22" color="#fa8c16"><WarningFilled /></el-icon>
          <div class="warning-inner">
            <h4>清单尚未全部完成</h4>
            <p>完成度: {{ activeExchange.checklistStats.completionRate }}% ({{ activeExchange.checklistStats.completed }}/{{ activeExchange.checklistStats.total }})</p>
          </div>
        </div>
        <div class="confirm-success-box" v-else-if="activeExchange.checklistStats && activeExchange.checklistStats.completionRate === 100">
          <el-icon size="28" color="#52c41a"><CircleCheckFilled /></el-icon>
          <div class="success-inner">
            <h4>所有清单已完成！</h4>
          </div>
        </div>
        <div class="confirm-hint">
          <p>确认完成后，交换状态将标记为已完成，双方各获得 50 技能积分。</p>
        </div>
      </div>
      <template #footer>
        <el-button @click="showConfirmFromDrawer = false">取消</el-button>
        <el-button type="primary" @click="doConfirmFromDrawer" :loading="confirmingFromDrawer">
          <el-icon><Check /></el-icon>确认完成
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, nextTick, computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { messageAPI, skillAPI, exchangeAPI, authAPI } from '../api'
import { useUserStore } from '../stores/user'
import { ElMessage } from 'element-plus'
import dayjs from 'dayjs'
import {
  ChatDotRound, Promotion, Connection, List, Switch,
  Upload, Download, WarningFilled, CircleCheckFilled, Check
} from '@element-plus/icons-vue'
import ExchangeChecklist from '../components/ExchangeChecklist.vue'
import ChecklistProgress from '../components/ChecklistProgress.vue'

const route = useRoute()
const router = useRouter()
const userStore = useUserStore()
const myId = userStore.user?.id

const conversations = ref([])
const messages = ref([])
const currentUserId = ref(null)
const currentUser = ref(null)
const messageInput = ref('')
const sending = ref(false)
const messagesContainer = ref(null)
const showExchangeDialog = ref(false)
const myTeachSkills = ref([])
const otherTeachSkills = ref([])
const exchangeForm = ref({
  teachSkill: '',
  learnSkill: ''
})

const showChecklistPanel = ref(false)
const activeExchange = ref(null)
const activeChecklistItems = ref([])
const chatUsers = ref({})
const conversationChecklistStats = ref({})
const showConfirmFromDrawer = ref(false)
const confirmingFromDrawer = ref(false)

const checklistPanelTitle = computed(() => {
  if (!currentUser.value) return '交换清单'
  return `清单 - ${currentUser.value.username}`
})

const activeChecklistDetailStats = computed(() => {
  const stats = {
    material: { total: 0, completed: 0 },
    practice: { total: 0, completed: 0 },
    homework: { total: 0, completed: 0 }
  }
  activeChecklistItems.value.forEach(item => {
    if (stats[item.type]) {
      stats[item.type].total++
      if (item.completed) stats[item.type].completed++
    }
  })
  return stats
})

onMounted(async () => {
  await loadConversations()
  if (route.params.userId) {
    await selectConversation(route.params.userId)
  }
})

watch(() => route.params.userId, async (newId) => {
  if (newId) {
    await selectConversation(newId)
  }
})

async function loadConversations() {
  try {
    const res = await messageAPI.getConversations()
    conversations.value = res.data
    const allExchangesRes = await exchangeAPI.getExchanges()
    const allExchanges = allExchangesRes.data
    conversationChecklistStats.value = {}
    res.data.forEach(conv => {
      const userExchanges = allExchanges.filter(e =>
        (e.initiatorId === myId && e.partnerId === conv.userId) ||
        (e.initiatorId === conv.userId && e.partnerId === myId)
      )
      const activeExc = userExchanges.find(e => e.status !== 'completed') || userExchanges[userExchanges.length - 1]
      if (activeExc?.checklistStats) {
        conversationChecklistStats.value[conv.userId] = activeExc.checklistStats
      }
    })
  } catch (e) {}
}

async function selectConversation(userId) {
  currentUserId.value = userId
  await loadMessages(userId)
  await loadConversations()

  const userRes = await authAPI.getUser(userId)
  currentUser.value = userRes.data
  chatUsers.value[myId] = { ...userStore.user, avatar: userStore.user?.avatar }
  chatUsers.value[userId] = userRes.data

  const [mySkills, otherSkills] = await Promise.all([
    skillAPI.getSkills({ userId: myId, type: 'teach' }),
    skillAPI.getSkills({ userId, type: 'teach' })
  ])
  myTeachSkills.value = mySkills.data
  otherTeachSkills.value = otherSkills.data

  await loadActiveExchange(userId)
}

async function loadActiveExchange(userId) {
  try {
    const allExchangesRes = await exchangeAPI.getExchanges()
    const userExchanges = allExchangesRes.data.filter(e =>
      (e.initiatorId === myId && e.partnerId === userId) ||
      (e.initiatorId === userId && e.partnerId === myId)
    )
    activeExchange.value = userExchanges.find(e => e.status !== 'completed') || userExchanges[userExchanges.length - 1] || null
    if (activeExchange.value) {
      try {
        const checklistRes = await exchangeAPI.getChecklists(activeExchange.value.id)
        activeChecklistItems.value = checklistRes.data
      } catch (e) {
        activeChecklistItems.value = []
      }
    } else {
      activeChecklistItems.value = []
    }
  } catch (e) {
    activeExchange.value = null
  }
}

async function loadMessages(userId) {
  try {
    const res = await messageAPI.getMessages(userId)
    messages.value = res.data
    await nextTick()
    scrollToBottom()
  } catch (e) {}
}

async function sendMessage() {
  if (!messageInput.value.trim()) return
  try {
    sending.value = true
    await messageAPI.sendMessage({
      receiverId: currentUserId.value,
      content: messageInput.value.trim()
    })
    messageInput.value = ''
    await loadMessages(currentUserId.value)
    await loadConversations()
  } catch (e) {
    ElMessage.error('发送失败')
  } finally {
    sending.value = false
  }
}

function scrollToBottom() {
  if (messagesContainer.value) {
    messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
  }
}

function formatTime(time) {
  const now = dayjs()
  const msgTime = dayjs(time)
  if (now.diff(msgTime, 'day') === 0) {
    return msgTime.format('HH:mm')
  } else if (now.diff(msgTime, 'day') === 1) {
    return '昨天'
  } else {
    return msgTime.format('MM-DD')
  }
}

function formatFullTime(time) {
  if (!time) return ''
  return dayjs(time).format('YYYY-MM-DD HH:mm')
}

function getProgressColor(rate) {
  if (rate === 100) return '#52c41a'
  if (rate >= 60) return '#1890ff'
  if (rate >= 30) return '#fa8c16'
  return '#ff4d4f'
}

async function createExchange() {
  try {
    const res = await exchangeAPI.createExchange({
      partnerId: currentUserId.value,
      skills: {
        teach: [exchangeForm.value.teachSkill],
        learn: [exchangeForm.value.learnSkill]
      }
    })
    ElMessage.success('交换请求已发送')
    showExchangeDialog.value = false
    exchangeForm.value = { teachSkill: '', learnSkill: '' }
    setTimeout(async () => {
      activeExchange.value = res.data
      if (activeExchange.value) {
        activeExchange.value.checklistStats = { total: 0, completed: 0, completionRate: 0 }
      }
      await loadConversations()
    }, 200)
  } catch (e) {
    ElMessage.error('发起失败')
  }
}

function getMyAvatar() {
  return userStore.user?.avatar || ''
}

function getMyTeachSkillsForExchange(exchange) {
  if (!exchange) return ''
  if (exchange.initiatorId === myId) {
    return exchange.skills?.teach?.join(', ') || '-'
  } else {
    return exchange.skills?.learn?.join(', ') || '-'
  }
}

function getMyLearnSkillsForExchange(exchange) {
  if (!exchange) return ''
  if (exchange.initiatorId === myId) {
    return exchange.skills?.learn?.join(', ') || '-'
  } else {
    return exchange.skills?.teach?.join(', ') || '-'
  }
}

async function onChecklistStatsUpdate(newStats) {
  if (activeExchange.value) {
    activeExchange.value.checklistStats = newStats
    try {
      const checklistRes = await exchangeAPI.getChecklists(activeExchange.value.id)
      activeChecklistItems.value = checklistRes.data
    } catch (e) {}
    await loadConversations()
  }
}

async function onChecklistPanelClose() {
  if (currentUserId.value) {
    await loadActiveExchange(currentUserId.value)
    await loadConversations()
  }
}

function confirmExchangeFromDrawer() {
  if (activeExchange.value?.confirmedBy?.includes(myId)) {
    ElMessage.info('您已确认过此交换')
    return
  }
  showConfirmFromDrawer.value = true
}

async function doConfirmFromDrawer() {
  if (!activeExchange.value) return
  try {
    confirmingFromDrawer.value = true
    await exchangeAPI.confirmExchange(activeExchange.value.id)
    ElMessage.success('确认成功')
    showConfirmFromDrawer.value = false
    if (currentUserId.value) {
      await loadActiveExchange(currentUserId.value)
      await loadConversations()
    }
  } catch (e) {
    ElMessage.error('确认失败')
  } finally {
    confirmingFromDrawer.value = false
  }
}
</script>

<style scoped>
.chat-page {
  height: calc(100vh - 120px);
}

.chat-layout {
  display: flex;
  height: 100%;
  background: white;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
}

.sidebar {
  width: 320px;
  border-right: 1px solid #eee;
  display: flex;
  flex-direction: column;
}

.sidebar-header {
  padding: 20px;
  border-bottom: 1px solid #eee;
}

.sidebar-header h3 {
  margin: 0;
  font-size: 18px;
  color: #333;
}

.conversation-list {
  flex: 1;
  overflow-y: auto;
}

.conversation-item {
  display: flex;
  gap: 12px;
  padding: 16px 20px;
  cursor: pointer;
  transition: all 0.2s;
  border-bottom: 1px solid #f5f5f5;
}

.conversation-item:hover {
  background: #f5f7ff;
}

.conversation-item.active {
  background: #eef1ff;
}

.conv-info {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.conv-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2px;
}

.conv-name {
  font-weight: 600;
  color: #333;
}

.conv-time {
  font-size: 12px;
  color: #999;
}

.conv-message {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.last-msg {
  color: #666;
  font-size: 13px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  max-width: 180px;
}

.unread-badge {
  margin-left: 8px;
}

.conv-checklist-mini {
  display: flex;
  align-items: center;
  gap: 4px;
  padding-top: 4px;
}

.mini-text {
  font-size: 11px;
  color: #667eea;
}

.chat-main {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.chat-header {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 24px;
  border-bottom: 1px solid #eee;
}

.header-user-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.chat-username {
  font-weight: 600;
  font-size: 16px;
  color: #333;
}

.header-checklist-status {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  color: #666;
}

.header-checklist-status :deep(.el-button) {
  padding: 0;
  font-size: 12px;
  height: auto;
  margin-left: 4px;
}

.status-text {
  font-weight: 500;
}

.header-actions {
  display: flex;
  gap: 8px;
}

.messages-container {
  flex: 1;
  padding: 24px;
  overflow-y: auto;
  background: #fafafa;
}

.message-row {
  display: flex;
  gap: 12px;
  margin-bottom: 16px;
}

.message-row.sent {
  flex-direction: row-reverse;
}

.chat-bubble {
  max-width: 60%;
  padding: 12px 16px;
  border-radius: 16px;
  line-height: 1.5;
  word-break: break-word;
}

.chat-bubble.sent {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-bottom-right-radius: 4px;
}

.chat-bubble.received {
  background: white;
  color: #333;
  border: 1px solid #f0f0f0;
  border-bottom-left-radius: 4px;
}

.chat-empty {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: #999;
  gap: 16px;
}

.chat-input-area {
  padding: 16px 24px;
  border-top: 1px solid #eee;
}

.checklist-drawer-wrapper {
  display: flex;
  flex-direction: column;
  gap: 16px;
  height: 100%;
}

.drawer-exchange-info {
  margin-bottom: 4px;
}

.exchange-info-card {
  background: linear-gradient(135deg, #f5f7ff 0%, #eef1ff 100%);
  border-radius: 10px;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.info-users-row {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
}

.info-icon {
  color: #667eea;
  font-size: 18px;
}

.info-skills-row {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.skill-badge {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 6px 12px;
  border-radius: 8px;
  font-size: 13px;
  font-weight: 500;
}

.skill-badge.teach {
  background: #e6f7ff;
  color: #1890ff;
}

.skill-badge.learn {
  background: #f6ffed;
  color: #52c41a;
}

.exchange-status-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}

.exchange-time-small {
  font-size: 11px;
  color: #999;
}

.drawer-divider {
  display: flex;
  align-items: center;
  color: #999;
  font-size: 12px;
  font-weight: 500;
  margin: 4px 0;
}

.drawer-divider::before,
.drawer-divider::after {
  content: '';
  flex: 1;
  height: 1px;
  background: #f0f0f0;
}

.drawer-divider::before {
  margin-right: 12px;
}

.drawer-divider::after {
  margin-left: 12px;
}

.drawer-footer {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
}

.drawer-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 16px;
  padding: 60px 20px;
  text-align: center;
}

.drawer-empty h4 {
  margin: 0;
  color: #333;
  font-size: 18px;
}

.drawer-empty p {
  margin: 0;
  color: #999;
  font-size: 14px;
}

.confirm-dialog-inner {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.confirm-warning-box {
  display: flex;
  gap: 12px;
  padding: 14px;
  background: #fff7e6;
  border-radius: 8px;
  border: 1px solid #ffe58f;
}

.warning-inner h4 {
  margin: 0 0 4px 0;
  color: #d48806;
  font-size: 14px;
}

.warning-inner p {
  margin: 0;
  color: #874d00;
  font-size: 13px;
}

.confirm-success-box {
  display: flex;
  gap: 12px;
  padding: 14px;
  background: #f6ffed;
  border-radius: 8px;
  border: 1px solid #b7eb8f;
  align-items: center;
}

.success-inner h4 {
  margin: 0;
  color: #389e0d;
  font-size: 14px;
}

.confirm-hint p {
  margin: 0;
  font-size: 13px;
  color: #666;
  line-height: 1.6;
  padding: 12px;
  background: #fafafa;
  border-radius: 6px;
}
</style>
