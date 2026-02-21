<template>
  <el-drawer
    v-model="drawerVisible"
    :title="`实时日志 - ${nickname}`"
    direction="rtl"
    :size="isMobile ? '100%' : '55%'"
    @open="handleOpen"
    @close="handleClose"
  >
    <div class="log-panel">
      <!-- 工具栏 -->
      <div class="log-toolbar">
        <el-input
          v-model="filterText"
          placeholder="搜索日志..."
          clearable
          size="small"
          style="width: 200px;"
          :prefix-icon="Search"
        />
        <el-checkbox v-model="autoScroll" size="small">自动滚动</el-checkbox>
        <el-button size="small" @click="clearLogs" plain>清空</el-button>
        <el-tag size="small" type="info">{{ filteredLogs.length }} 条</el-tag>
      </div>

      <!-- 日志列表 -->
      <div class="log-list" ref="logListRef">
        <div
          v-for="(log, idx) in filteredLogs"
          :key="idx"
          class="log-entry"
          :class="'log-' + log.level"
        >
          <span class="log-time">{{ log.time || formatTs(log.ts) }}</span>
          <span class="log-level-badge" :class="'badge-' + log.level">{{ getLevelLabel(log.level) }}</span>
          <span class="log-tag">{{ log.icon || '' }} {{ log.tag }}</span>
          <span class="log-msg">{{ log.msg }}</span>
        </div>
        <div v-if="filteredLogs.length === 0" class="log-empty">
          暂无日志
        </div>
      </div>
    </div>
  </el-drawer>
</template>

<script setup>
import { ref, computed, watch, nextTick, onUnmounted } from 'vue'
import { Search } from '@element-plus/icons-vue'
import { subscribeLogs, unsubscribeLogs, onEvent, offEvent } from '../socket/index.js'
import { getAccountLogs } from '../api/index.js'

const props = defineProps({
  visible: Boolean,
  uin: String,
  nickname: String,
})

// 移动端检测
const isMobile = ref(window.innerWidth <= 768)
function handleResize() {
  isMobile.value = window.innerWidth <= 768
}
window.addEventListener('resize', handleResize)

const emit = defineEmits(['update:visible'])

const drawerVisible = computed({
  get: () => props.visible,
  set: (v) => emit('update:visible', v),
})

const logs = ref([])
const filterText = ref('')
const autoScroll = ref(true)
const logListRef = ref(null)

const filteredLogs = computed(() => {
  if (!filterText.value) return logs.value
  const kw = filterText.value.toLowerCase()
  return logs.value.filter(l =>
    (l.tag && l.tag.toLowerCase().includes(kw)) ||
    (l.msg && l.msg.toLowerCase().includes(kw))
  )
})

// 实时日志监听
function onBotLog(entry) {
  if (entry.userId !== props.uin) return
  logs.value.push(entry)
  if (logs.value.length > 2000) logs.value.splice(0, logs.value.length - 1500) // 保持最多 1500
  if (autoScroll.value) scrollToBottom()
}

function onLogsHistory(data) {
  if (data.uin !== props.uin) return
  logs.value = data.logs || []
  nextTick(scrollToBottom)
}

function scrollToBottom() {
  nextTick(() => {
    if (logListRef.value) {
      logListRef.value.scrollTop = logListRef.value.scrollHeight
    }
  })
}

async function handleOpen() {
  logs.value = []
  // 先加载历史日志
  try {
    const res = await getAccountLogs(props.uin, 500)
    logs.value = res.data || []
  } catch (e) { /* ignore */ }

  // 订阅实时日志
  subscribeLogs(props.uin)
  onEvent('bot:log', onBotLog)
  onEvent('logs:history', onLogsHistory)
  nextTick(scrollToBottom)
}

function handleClose() {
  unsubscribeLogs(props.uin)
  offEvent('bot:log', onBotLog)
  offEvent('logs:history', onLogsHistory)
}

function clearLogs() {
  logs.value = []
}

function formatTs(ts) {
  if (!ts) return ''
  return new Date(ts).toLocaleTimeString()
}

function getLevelLabel(level) {
  if (level === 'error') return 'ERR'
  if (level === 'warn') return 'WRN'
  return 'INF'
}

onUnmounted(() => {
  offEvent('bot:log', onBotLog)
  offEvent('logs:history', onLogsHistory)
  window.removeEventListener('resize', handleResize)
})
</script>

<style scoped>
.log-panel {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.log-toolbar {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 0 0 12px 0;
  border-bottom: 1px solid #ebeef5;
  flex-shrink: 0;
  flex-wrap: wrap;
}

.log-list {
  flex: 1;
  overflow-y: auto;
  font-family: 'Menlo', 'Consolas', 'Monaco', monospace;
  font-size: 12px;
  line-height: 1.8;
  padding: 8px 0;
  background: #1e1e1e;
  border-radius: 6px;
  margin-top: 12px;
  padding: 12px;
}

.log-entry {
  white-space: pre-wrap;
  word-break: break-all;
  display: flex;
  align-items: baseline;
  gap: 6px;
  padding: 1px 0;
  border-bottom: 1px solid rgba(48, 54, 61, 0.3);
}

.log-entry:last-child {
  border-bottom: none;
}

.log-time {
  color: #6e7681;
  margin-right: 2px;
  font-size: 11px;
  min-width: 80px;
  flex-shrink: 0;
}

.log-level-badge {
  display: inline-block;
  font-size: 9px;
  font-weight: 700;
  padding: 0 3px;
  border-radius: 3px;
  flex-shrink: 0;
  min-width: 26px;
  text-align: center;
  line-height: 16px;
}

.badge-info {
  background: rgba(56, 139, 253, 0.15);
  color: #58a6ff;
}

.badge-warn {
  background: rgba(210, 153, 34, 0.15);
  color: #d29922;
}

.badge-error {
  background: rgba(248, 81, 73, 0.15);
  color: #f85149;
}

.log-tag {
  color: #79c0ff;
  margin-right: 2px;
  font-weight: 600;
  min-width: 50px;
  flex-shrink: 0;
}

.log-msg {
  color: #d4d4d4;
  flex: 1;
}

.log-warn .log-tag {
  color: #d29922;
}

.log-warn .log-msg {
  color: #d29922;
}

.log-error .log-tag {
  color: #f85149;
}

.log-error .log-msg {
  color: #f85149;
}

.log-empty {
  color: #6a6a6a;
  text-align: center;
  padding: 40px 0;
}

@media (max-width: 768px) {
  .log-toolbar {
    gap: 6px;
  }

  .log-toolbar .el-input {
    width: 100% !important;
    flex: 1 1 100%;
  }

  .log-list {
    font-size: 11px;
    padding: 8px;
  }
}
</style>
