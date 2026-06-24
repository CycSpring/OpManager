<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'
import { Dialogs } from '@wailsio/runtime'
import { PortService } from '../bindings/github.com/fengfengzhidao/port_lite'
import type { PortEntry, PortListResult } from '../bindings/github.com/fengfengzhidao/port_lite/models'

type Protocol = 'ALL' | 'TCP' | 'UDP'

const ports = ref<PortEntry[]>([])
const stats = ref<PortListResult>({
  ports: [],
  tcpCount: 0,
  udpCount: 0,
  listeningCount: 0,
  processCount: 0,
  warnings: [],
})
const loading = ref(false)
const killingPid = ref<number | null>(null)
const error = ref('')
const message = ref('')
const query = ref('')
const portFilter = ref('')
const ipFilter = ref('')
const protocolFilter = ref<Protocol>('ALL')
const stateFilter = ref('ALL')

// 状态选项：根据当前协议动态计算，始终排除 "-"
const stateOptions = computed(() => {
  const states = new Set<string>()
  const proto = protocolFilter.value
  for (const item of ports.value) {
    if (proto !== 'ALL' && item.protocol !== proto) continue
    if (item.state && item.state !== '-') states.add(item.state)
  }
  return ['ALL', ...Array.from(states).filter(s => s !== '-').sort()]
})

// 切协议时，自动重置不在新选项中的状态
watch(protocolFilter, () => {
  if (!stateOptions.value.includes(stateFilter.value)) {
    stateFilter.value = 'ALL'
  }
})

const filteredPorts = computed(() => {
  const keyword = query.value.trim().toLowerCase()
  const port = portFilter.value.trim()
  const ip = ipFilter.value.trim().toLowerCase()
  const proto = protocolFilter.value
  const st = stateFilter.value

  return ports.value.filter((item) => {
    if (proto !== 'ALL' && item.protocol !== proto) return false
    if (st !== 'ALL' && (item.state === '-' || item.state !== st)) return false
    // 端口：精确匹配
    if (port && String(item.localPort) !== port) return false
    // IP：模糊匹配
    if (ip && !item.localAddr.toLowerCase().includes(ip) && !item.remoteAddr.toLowerCase().includes(ip)) return false
    // 搜索：全字段模糊匹配
    if (keyword) {
      return [
        item.protocol,
        item.localAddr,
        String(item.localPort),
        item.remoteAddr,
        String(item.remotePort || ''),
        item.state,
        String(item.pid),
        item.process,
        item.path,
      ].some((part) => part.toLowerCase().includes(keyword))
    }
    return true
  })
})

// 清空所有筛选
function clearFilters() {
  query.value = ''
  portFilter.value = ''
  ipFilter.value = ''
  protocolFilter.value = 'ALL'
  stateFilter.value = 'ALL'
}

const lastUpdated = ref('')

function formatNumber(value: number) {
  return new Intl.NumberFormat('zh-CN').format(value)
}

function endpoint(addr: string, port: number) {
  if (!addr && !port) return '-'
  if (!port) return addr || '-'
  return `${addr}:${port}`
}

function canKill(item: PortEntry) {
  return item.pid !== 0 && item.pid !== 4
}

async function refreshPorts(clearNotice = true) {
  if (loading.value) return

  loading.value = true
  if (clearNotice) {
    error.value = ''
    message.value = ''
  }

  try {
    const result = await PortService.ListPorts()
    ports.value = result.ports
    stats.value = result
    lastUpdated.value = new Date().toLocaleTimeString('zh-CN', { hour12: false })
  } catch (err) {
    error.value = err instanceof Error ? err.message : String(err)
  } finally {
    loading.value = false
  }
}

async function killProcess(item: PortEntry) {
  if (!canKill(item) || killingPid.value !== null) return

  const answer = await Dialogs.Question({
    Title: '确认结束进程',
    Message: `端口 ${item.localPort} 当前由 ${item.process || `PID ${item.pid}`} 占用。\n\n结束的是进程，不是单个端口。确认继续吗？`,
    Buttons: [
      { Label: 'No', IsCancel: true },
      { Label: 'Yes', IsDefault: true },
    ],
  })

  if (answer !== 'Yes') return

  killingPid.value = item.pid
  error.value = ''
  message.value = ''

  try {
    const result = await PortService.KillProcess(item.pid)
    message.value = result.message || `已结束 PID ${item.pid}`
    await refreshPorts(false)

    const remaining = ports.value.find((port) => {
      return port.protocol === item.protocol && port.localAddr === item.localAddr && port.localPort === item.localPort
    })
    if (remaining) {
      if (remaining.pid === item.pid) {
        error.value = `PID ${item.pid} 已发送结束请求，但端口 ${item.localPort} 仍然存在，可能需要管理员权限或进程没有退出。`
      } else {
        message.value = `已结束 PID ${item.pid}，但端口 ${item.localPort} 又被 ${remaining.process || `PID ${remaining.pid}`} 占用。`
      }
    }
  } catch (err) {
    error.value = err instanceof Error ? err.message : String(err)
  } finally {
    killingPid.value = null
  }
}

onMounted(refreshPorts)
</script>

<template>
  <main class="app-shell">
    <header class="topbar">
      <div>
        <p class="eyebrow">Windows Port Watcher</p>
        <h1>PortLite</h1>
      </div>
      <button class="refresh-button" type="button" :disabled="loading" @click="() => refreshPorts()">
        {{ loading ? '刷新中...' : '刷新' }}
      </button>
    </header>

    <section class="stats-grid" aria-label="端口统计">
      <article class="stat-card">
        <span>TCP</span>
        <strong>{{ formatNumber(stats.tcpCount) }}</strong>
      </article>
      <article class="stat-card">
        <span>UDP</span>
        <strong>{{ formatNumber(stats.udpCount) }}</strong>
      </article>
      <article class="stat-card">
        <span>监听端口</span>
        <strong>{{ formatNumber(stats.listeningCount) }}</strong>
      </article>
      <article class="stat-card">
        <span>进程</span>
        <strong>{{ formatNumber(stats.processCount) }}</strong>
      </article>
    </section>

    <section class="toolbar" aria-label="筛选端口">
      <label>
        <span>端口</span>
        <input v-model="portFilter" type="search" placeholder="精确端口号" />
      </label>

      <label>
        <span>协议</span>
        <select v-model="protocolFilter">
          <option value="ALL">全部</option>
          <option value="TCP">TCP</option>
          <option value="UDP">UDP</option>
        </select>
      </label>

      <label>
        <span>状态</span>
        <select v-model="stateFilter">
          <option v-for="state in stateOptions" :key="state" :value="state">
            {{ state === 'ALL' ? '全部' : state }}
          </option>
        </select>
      </label>

      <label>
        <span>IP</span>
        <input v-model="ipFilter" type="search" placeholder="本地或远程 IP" />
      </label>

      <label>
        <span>搜索</span>
        <input v-model="query" type="search" placeholder="进程名、PID、路径" />
      </label>

      <div class="toolbar-actions">
        <p class="count-text">
          当前显示 {{ formatNumber(filteredPorts.length) }} 条
          <span v-if="lastUpdated">，{{ lastUpdated }} 更新</span>
        </p>
        <button
          v-if="query || portFilter || ipFilter || protocolFilter !== 'ALL' || stateFilter !== 'ALL'"
          class="clear-button"
          type="button"
          @click="clearFilters"
        >清空筛选</button>
      </div>
    </section>

    <p v-if="message" class="notice success">{{ message }}</p>
    <p v-if="error" class="notice error">{{ error }}</p>
    <p v-for="warning in stats.warnings" :key="warning" class="notice warning">{{ warning }}</p>

    <section class="table-wrap">
      <table>
        <thead>
          <tr>
            <th>协议</th>
            <th>本地端口</th>
            <th>本地地址</th>
            <th>远程地址</th>
            <th>状态</th>
            <th>进程</th>
            <th>PID</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-if="!loading && filteredPorts.length === 0">
            <td colspan="8" class="empty">没有匹配的端口，请尝试调整筛选条件</td>
          </tr>
          <tr
            v-for="(item, index) in filteredPorts"
            :key="`${item.protocol}-${item.localAddr}-${item.localPort}-${item.remoteAddr}-${item.remotePort}-${item.state}-${item.pid}-${index}`"
          >
            <td>
              <span class="protocol" :class="item.protocol.toLowerCase()">{{ item.protocol }}</span>
            </td>
            <td class="port">{{ item.localPort }}</td>
            <td>{{ endpoint(item.localAddr, item.localPort) }}</td>
            <td>{{ item.protocol === 'UDP' ? '-' : endpoint(item.remoteAddr, item.remotePort) }}</td>
            <td>
              <span class="state" :class="item.state.toLowerCase()">{{ item.state }}</span>
            </td>
            <td>
              <div class="process-cell">
                <strong>{{ item.process || `PID ${item.pid}` }}</strong>
                <small v-if="item.path">{{ item.path }}</small>
              </div>
            </td>
            <td>{{ item.pid }}</td>
            <td>
              <button
                class="kill-button"
                type="button"
                :disabled="!canKill(item) || killingPid !== null"
                @click="killProcess(item)"
              >
                {{ killingPid === item.pid ? '处理中' : '结束进程' }}
              </button>
            </td>
          </tr>
        </tbody>
      </table>
    </section>
  </main>
</template>

<style scoped>
:global(*) {
  box-sizing: border-box;
}

:global(body) {
  margin: 0;
  min-width: 980px;
  color: #172033;
  background: #eef2f6;
  font-family:
    Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI",
    "Microsoft YaHei", sans-serif;
}

button,
input,
select {
  font: inherit;
}

.app-shell {
  min-height: 100vh;
  padding: 24px;
}

.topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 18px;
}

.eyebrow {
  margin: 0 0 4px;
  color: #657086;
  font-size: 13px;
  font-weight: 700;
  text-transform: uppercase;
}

h1 {
  margin: 0;
  font-size: 34px;
  line-height: 1.1;
  letter-spacing: 0;
}

.refresh-button,
.kill-button {
  border: 0;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 700;
}

.refresh-button {
  min-width: 92px;
  padding: 11px 18px;
  color: #fff;
  background: #126c5b;
}

.refresh-button:disabled,
.kill-button:disabled {
  cursor: not-allowed;
  opacity: 0.55;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 12px;
  margin-bottom: 16px;
}

.stat-card {
  padding: 16px;
  border: 1px solid #d9e0ea;
  border-radius: 8px;
  background: #fff;
}

.stat-card span {
  display: block;
  margin-bottom: 8px;
  color: #6b7280;
  font-size: 13px;
}

.stat-card strong {
  font-size: 28px;
  line-height: 1;
}

.toolbar {
  display: grid;
  grid-template-columns: 120px 100px 140px minmax(140px, 1fr) minmax(160px, 1fr) auto;
  align-items: end;
  gap: 12px;
  margin-bottom: 12px;
  padding: 14px;
  border: 1px solid #d9e0ea;
  border-radius: 8px;
  background: #fff;
}

label span {
  display: block;
  margin-bottom: 6px;
  color: #667085;
  font-size: 12px;
  font-weight: 700;
}

input,
select {
  width: 100%;
  height: 38px;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  padding: 0 10px;
  color: #1f2937;
  background: #f8fafc;
}

.toolbar-actions {
  display: flex;
  align-items: end;
  gap: 10px;
}

.count-text {
  margin: 0;
  color: #667085;
  font-size: 13px;
  white-space: nowrap;
  line-height: 38px;
}

.clear-button {
  height: 38px;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  padding: 0 12px;
  color: #475569;
  background: #f8fafc;
  cursor: pointer;
  font-weight: 600;
  font-size: 13px;
}

.clear-button:hover {
  background: #e2e8f0;
}

.notice {
  margin: 0 0 10px;
  border-radius: 8px;
  padding: 10px 12px;
  font-size: 14px;
}

.success {
  color: #105b4c;
  background: #dcfce7;
}

.error {
  color: #9f1239;
  background: #ffe4e6;
}

.warning {
  color: #854d0e;
  background: #fef3c7;
}

.table-wrap {
  overflow: auto;
  border: 1px solid #d9e0ea;
  border-radius: 8px;
  background: #fff;
}

table {
  width: 100%;
  min-width: 1040px;
  border-collapse: collapse;
}

th,
td {
  border-bottom: 1px solid #edf1f5;
  padding: 12px 14px;
  text-align: left;
  vertical-align: middle;
  font-size: 14px;
}

th {
  position: sticky;
  top: 0;
  z-index: 1;
  color: #667085;
  background: #f8fafc;
  font-size: 12px;
  font-weight: 800;
}

tr:hover td {
  background: #f8fbfd;
}

.protocol,
.state {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 52px;
  height: 24px;
  border-radius: 999px;
  padding: 0 8px;
  font-size: 12px;
  font-weight: 800;
}

.protocol.tcp {
  color: #075985;
  background: #e0f2fe;
}

.protocol.udp {
  color: #6d28d9;
  background: #ede9fe;
}

.state {
  color: #475569;
  background: #f1f5f9;
}

.state.listening {
  color: #047857;
  background: #d1fae5;
}

.state.established {
  color: #075985;
  background: #e0f2fe;
}

.state.time-wait {
  color: #92400e;
  background: #fef3c7;
}

.state.close-wait {
  color: #9f1239;
  background: #ffe4e6;
}

.port {
  color: #111827;
  font-weight: 800;
}

.process-cell {
  max-width: 340px;
}

.process-cell strong,
.process-cell small {
  display: block;
}

.process-cell small {
  overflow: hidden;
  color: #7b8496;
  font-size: 12px;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.kill-button {
  min-width: 86px;
  height: 32px;
  color: #9f1239;
  background: #ffe4e6;
}

.empty {
  height: 180px;
  color: #7b8496;
  text-align: center;
}
</style>
