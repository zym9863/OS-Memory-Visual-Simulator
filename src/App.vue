<template>
  <div id="app">
    <header class="header">
      <h1>操作系统内存可视化模拟器</h1>
      <p>OS Memory Visual Simulator</p>
    </header>
    
    <div class="main-container">
      <!-- 控制面板 -->
      <div class="control-panel">
        <div class="config-section">
          <h3>系统配置</h3>
          <div class="config-item">
            <label>总内存大小 (KB):</label>
            <input 
              v-model.number="memorySize" 
              type="number" 
              min="100" 
              max="2048" 
              :disabled="isRunning"
            />
          </div>
          
          <div class="config-item">
            <label>分配算法:</label>
            <select v-model="allocationAlgorithm" :disabled="isRunning">
              <option value="first-fit">首次适应 (First Fit)</option>
              <option value="best-fit">最佳适应 (Best Fit)</option>
              <option value="worst-fit">最坏适应 (Worst Fit)</option>
              <option value="next-fit">循环首次适应 (Next Fit)</option>
            </select>
          </div>
        </div>

        <div class="job-section">
          <h3>作业管理</h3>
          <div class="job-input">
            <div class="input-group">
              <label>作业名称:</label>
              <input v-model="newJob.name" type="text" placeholder="例: P1" />
            </div>
            <div class="input-group">
              <label>大小 (KB):</label>
              <input v-model.number="newJob.size" type="number" min="1" max="500" />
            </div>
            <button @click="addJob" :disabled="!newJob.name || !newJob.size">
              添加作业
            </button>
          </div>
          
          <div class="job-queue">
            <h4>作业队列</h4>
            <div class="job-list">
              <div 
                v-for="(job, index) in jobQueue" 
                :key="index"
                class="job-item"
                :class="{ 'current': index === currentJobIndex }"
              >
                <span>{{ job.name }} ({{ job.size }}KB)</span>
                <button @click="removeJob(index)" :disabled="isRunning">删除</button>
              </div>
            </div>
          </div>
        </div>

        <div class="control-buttons">
          <button @click="startSimulation" :disabled="jobQueue.length === 0 || isRunning">
            开始模拟
          </button>
          <button @click="nextStep" :disabled="!isRunning || currentJobIndex >= jobQueue.length">
            下一步
          </button>
          <button @click="pauseSimulation" :disabled="!isRunning">
            {{ isPaused ? '继续' : '暂停' }}
          </button>
          <button @click="resetSimulation">
            重置
          </button>
        </div>

        <div class="allocated-jobs">
          <h4>已分配作业</h4>
          <div class="allocated-list">
            <div 
              v-for="job in allocatedJobs" 
              :key="job.id"
              class="allocated-item"
              :style="{ backgroundColor: job.color }"
            >
              <span>{{ job.name }} ({{ job.size }}KB)</span>
              <button @click="releaseJob(job.id)">释放</button>
            </div>
          </div>
        </div>
      </div>

      <!-- 内存可视化区域 -->
      <div class="visualization-area">
        <div class="memory-info">
          <h3>内存状态</h3>
          <div class="stats">
            <div class="stat-item">
              <span>总内存: {{ memorySize }}KB</span>
            </div>
            <div class="stat-item">
              <span>已用内存: {{ usedMemory }}KB</span>
            </div>
            <div class="stat-item">
              <span>空闲内存: {{ freeMemory }}KB</span>
            </div>
            <div class="stat-item">
              <span>内存利用率: {{ utilizationRate }}%</span>
            </div>
          </div>
        </div>

        <div class="memory-blocks">
          <div 
            v-for="(block, index) in memoryBlocks" 
            :key="index"
            class="memory-block"
            :class="{ 
              'allocated': block.allocated, 
              'free': !block.allocated,
              'highlight': block.highlight 
            }"
            :style="{ 
              height: (block.size / memorySize * 400) + 'px',
              backgroundColor: block.allocated ? block.color : '#e0e0e0'
            }"
            @click="block.allocated && showJobInfo(block)"
          >
            <div class="block-info">
              <div class="block-label">
                {{ block.allocated ? block.jobName : '空闲' }}
              </div>
              <div class="block-size">
                {{ block.size }}KB
              </div>
              <div v-if="block.allocated" class="block-address">
                {{ block.startAddress }}-{{ block.startAddress + block.size - 1 }}
              </div>
            </div>
          </div>
        </div>

        <!-- 算法执行日志 -->
        <div class="execution-log">
          <h4>执行日志</h4>
          <div class="log-content">
            <div 
              v-for="(log, index) in executionLogs" 
              :key="index"
              class="log-item"
              :class="log.type"
            >
              <span class="log-time">{{ log.time }}</span>
              <span class="log-message">{{ log.message }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 高级功能组件 -->
    <MemoryAllocator
      ref="memoryAllocatorRef"
      :memory-blocks="memoryBlocks"
      :allocated-jobs="allocatedJobs"
      :memory-size="memorySize"
      :current-algorithm="allocationAlgorithm"
      :is-running="isRunning"
      @load-scenario="handleLoadScenario"
      @start-auto="handleStartAuto"
      @stop-auto="handleStopAuto"
      @restore-state="handleRestoreState"
    />
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, nextTick } from 'vue'
import MemoryAllocator from './components/MemoryAllocator.vue'

// 数据类型定义
interface Job {
  id: number
  name: string
  size: number
  color?: string
}

interface MemoryBlock {
  startAddress: number
  size: number
  allocated: boolean
  jobId?: number
  jobName?: string
  color?: string
  highlight?: boolean
}

interface ExecutionLog {
  time: string
  message: string
  type: 'success' | 'error' | 'info'
}

interface MemoryState {
  action: string
  timestamp: string
  memoryBlocks: MemoryBlock[]
  allocatedJobs: Job[]
  memorySize: number
  algorithm: string
}

interface TestScenario {
  name: string
  description: string
  jobs: { name: string; size: number }[]
  memorySize?: number
  algorithm?: string
}

interface AutoModeParams {
  scenario: TestScenario
  speed: number
}

// 响应式数据
const memorySize = ref(1024)
const allocationAlgorithm = ref('first-fit')
const isRunning = ref(false)
const isPaused = ref(false)
const currentJobIndex = ref(0)
const nextFitPointer = ref(0)

const newJob = ref({ name: '', size: 0 })
const jobQueue = ref<Job[]>([])
const allocatedJobs = ref<Job[]>([])
const memoryBlocks = ref<MemoryBlock[]>([])
const executionLogs = ref<ExecutionLog[]>([])

// 组件引用和自动模式
const memoryAllocatorRef = ref<InstanceType<typeof MemoryAllocator> | null>(null)
const autoTimer = ref<NodeJS.Timeout | null>(null)
const isAutoMode = ref(false)

let jobIdCounter = 1

// 颜色池
const colors = [
  '#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4', '#FFEAA7',
  '#DDA0DD', '#98D8C8', '#F7DC6F', '#BB8FCE', '#85C1E9'
]

// 计算属性
const usedMemory = computed(() => {
  return memoryBlocks.value
    .filter(block => block.allocated)
    .reduce((sum, block) => sum + block.size, 0)
})

const freeMemory = computed(() => memorySize.value - usedMemory.value)

const utilizationRate = computed(() => {
  return Math.round((usedMemory.value / memorySize.value) * 100)
})

// 初始化内存
const initializeMemory = () => {
  memoryBlocks.value = [{
    startAddress: 0,
    size: memorySize.value,
    allocated: false
  }]
  allocatedJobs.value = []
  currentJobIndex.value = 0
  nextFitPointer.value = 0
  jobIdCounter = 1
  addLog('系统初始化完成', 'info')
}

// 添加作业到队列
const addJob = () => {
  if (newJob.value.name && newJob.value.size > 0) {
    jobQueue.value.push({
      id: jobIdCounter++,
      name: newJob.value.name,
      size: newJob.value.size
    })
    newJob.value = { name: '', size: 0 }
    addLog(`作业 ${jobQueue.value[jobQueue.value.length - 1].name} 已添加到队列`, 'info')
  }
}

// 移除作业
const removeJob = (index: number) => {
  const job = jobQueue.value[index]
  jobQueue.value.splice(index, 1)
  addLog(`作业 ${job.name} 已从队列移除`, 'info')
}

// 开始模拟
const startSimulation = () => {
  isRunning.value = true
  isPaused.value = false
  currentJobIndex.value = 0
  addLog('开始内存分配模拟', 'info')
}

// 下一步
const nextStep = () => {
  if (currentJobIndex.value < jobQueue.value.length && !isPaused.value) {
    const job = jobQueue.value[currentJobIndex.value]
    allocateMemory(job)
    currentJobIndex.value++
    
    if (currentJobIndex.value >= jobQueue.value.length) {
      addLog('所有作业处理完成', 'info')
    }
  }
}

// 暂停/继续
const pauseSimulation = () => {
  isPaused.value = !isPaused.value
  addLog(isPaused.value ? '模拟已暂停' : '模拟已继续', 'info')
}

// 重置模拟
const resetSimulation = () => {
  isRunning.value = false
  isPaused.value = false
  initializeMemory()
  executionLogs.value = []
  addLog('系统已重置', 'info')
}

// 内存分配算法
const allocateMemory = (job: Job) => {
  let allocated = false
  
  switch (allocationAlgorithm.value) {
    case 'first-fit':
      allocated = firstFitAllocation(job)
      break
    case 'best-fit':
      allocated = bestFitAllocation(job)
      break
    case 'worst-fit':
      allocated = worstFitAllocation(job)
      break
    case 'next-fit':
      allocated = nextFitAllocation(job)
      break
  }
  
  if (allocated) {
    addLog(`作业 ${job.name} 分配成功`, 'success')
  } else {
    addLog(`作业 ${job.name} 分配失败：内存不足`, 'error')
  }
}

// 首次适应算法
const firstFitAllocation = (job: Job): boolean => {
  for (let i = 0; i < memoryBlocks.value.length; i++) {
    const block = memoryBlocks.value[i]
    if (!block.allocated && block.size >= job.size) {
      allocateBlock(i, job)
      return true
    }
  }
  return false
}

// 最佳适应算法
const bestFitAllocation = (job: Job): boolean => {
  let bestIndex = -1
  let bestSize = Infinity
  
  for (let i = 0; i < memoryBlocks.value.length; i++) {
    const block = memoryBlocks.value[i]
    if (!block.allocated && block.size >= job.size && block.size < bestSize) {
      bestIndex = i
      bestSize = block.size
    }
  }
  
  if (bestIndex !== -1) {
    allocateBlock(bestIndex, job)
    return true
  }
  return false
}

// 最坏适应算法
const worstFitAllocation = (job: Job): boolean => {
  let worstIndex = -1
  let worstSize = -1
  
  for (let i = 0; i < memoryBlocks.value.length; i++) {
    const block = memoryBlocks.value[i]
    if (!block.allocated && block.size >= job.size && block.size > worstSize) {
      worstIndex = i
      worstSize = block.size
    }
  }
  
  if (worstIndex !== -1) {
    allocateBlock(worstIndex, job)
    return true
  }
  return false
}

// 循环首次适应算法
const nextFitAllocation = (job: Job): boolean => {
  const startIndex = nextFitPointer.value
  
  // 从当前指针位置开始查找
  for (let i = 0; i < memoryBlocks.value.length; i++) {
    const index = (startIndex + i) % memoryBlocks.value.length
    const block = memoryBlocks.value[index]
    
    if (!block.allocated && block.size >= job.size) {
      allocateBlock(index, job)
      nextFitPointer.value = index
      return true
    }
  }
  return false
}

// 分配内存块
const allocateBlock = (blockIndex: number, job: Job) => {
  const block = memoryBlocks.value[blockIndex]
  const color = colors[allocatedJobs.value.length % colors.length]
  
  // 添加到已分配列表
  allocatedJobs.value.push({
    ...job,
    color
  })
  
  // 如果块大小正好等于作业大小，直接分配
  if (block.size === job.size) {
    block.allocated = true
    block.jobId = job.id
    block.jobName = job.name
    block.color = color
  } else {
    // 分割内存块
    const allocatedBlock: MemoryBlock = {
      startAddress: block.startAddress,
      size: job.size,
      allocated: true,
      jobId: job.id,
      jobName: job.name,
      color
    }
    
    const remainingBlock: MemoryBlock = {
      startAddress: block.startAddress + job.size,
      size: block.size - job.size,
      allocated: false
    }
    
    // 替换原块
    memoryBlocks.value.splice(blockIndex, 1, allocatedBlock, remainingBlock)
  }
}

// 释放作业
const releaseJob = (jobId: number) => {
  const jobIndex = allocatedJobs.value.findIndex(job => job.id === jobId)
  if (jobIndex === -1) return
  
  const job = allocatedJobs.value[jobIndex]
  
  // 找到对应的内存块并释放
  for (let i = 0; i < memoryBlocks.value.length; i++) {
    if (memoryBlocks.value[i].jobId === jobId) {
      memoryBlocks.value[i].allocated = false
      memoryBlocks.value[i].jobId = undefined
      memoryBlocks.value[i].jobName = undefined
      memoryBlocks.value[i].color = undefined
      break
    }
  }
  
  // 合并相邻的空闲块
  mergeAdjacentFreeBlocks()
  
  // 从已分配列表移除
  allocatedJobs.value.splice(jobIndex, 1)
  
  addLog(`作业 ${job.name} 已释放`, 'success')
}

// 合并相邻空闲块
const mergeAdjacentFreeBlocks = () => {
  for (let i = 0; i < memoryBlocks.value.length - 1; i++) {
    const current = memoryBlocks.value[i]
    const next = memoryBlocks.value[i + 1]
    
    if (!current.allocated && !next.allocated && 
        current.startAddress + current.size === next.startAddress) {
      // 合并两个块
      current.size += next.size
      memoryBlocks.value.splice(i + 1, 1)
      i-- // 重新检查当前位置
    }
  }
}

// 显示作业信息
const showJobInfo = (block: MemoryBlock) => {
  addLog(`作业 ${block.jobName}: 大小=${block.size}KB, 地址=${block.startAddress}-${block.startAddress + block.size - 1}`, 'info')
}

// 添加日志
const addLog = (message: string, type: 'success' | 'error' | 'info') => {
  const now = new Date()
  const time = now.toLocaleTimeString()
  executionLogs.value.unshift({ time, message, type })
  
  // 限制日志数量
  if (executionLogs.value.length > 50) {
    executionLogs.value = executionLogs.value.slice(0, 50)
  }
}

// MemoryAllocator 组件事件处理
const handleLoadScenario = (scenario: TestScenario) => {
  // 清空当前作业队列
  jobQueue.value = []
  
  // 如果场景包含内存大小和算法设置，应用它们
  if (scenario.memorySize) {
    memorySize.value = scenario.memorySize
  }
  if (scenario.algorithm) {
    allocationAlgorithm.value = scenario.algorithm
  }
  
  // 添加场景中的作业到队列
  scenario.jobs.forEach(job => {
    jobQueue.value.push({
      id: jobIdCounter++,
      name: job.name,
      size: job.size
    })
  })
  
  addLog(`已加载场景: ${scenario.name}`, 'info')
  
  // 重置模拟状态
  if (!isRunning.value) {
    initializeMemory()
  }
}

const handleStartAuto = ({ scenario, speed }: AutoModeParams) => {
  if (!scenario || scenario.jobs.length === 0) return
  
  isAutoMode.value = true
  isRunning.value = true
  isPaused.value = false
  currentJobIndex.value = 0
  
  addLog(`开始自动模式: ${scenario.name}`, 'info')
  
  // 自动执行下一步
  const executeNextStep = () => {
    if (!isAutoMode.value || isPaused.value) return
    
    if (currentJobIndex.value < jobQueue.value.length) {
      const job = jobQueue.value[currentJobIndex.value]
      allocateMemory(job)
      
      // 保存状态到历史记录
      if (memoryAllocatorRef.value) {
        memoryAllocatorRef.value.saveCurrentState(`分配作业 ${job.name}`)
      }
      
      currentJobIndex.value++
      
      if (currentJobIndex.value < jobQueue.value.length) {
        autoTimer.value = setTimeout(executeNextStep, speed)
      } else {
        addLog('自动模式完成', 'info')
        isAutoMode.value = false
      }
    } else {
      isAutoMode.value = false
    }
  }
  
  // 开始执行
  autoTimer.value = setTimeout(executeNextStep, speed)
}

const handleStopAuto = () => {
  isAutoMode.value = false
  if (autoTimer.value) {
    clearTimeout(autoTimer.value)
    autoTimer.value = null
  }
  addLog('自动模式已停止', 'info')
}

const handleRestoreState = (state: MemoryState) => {
  // 恢复内存状态
  memoryBlocks.value = JSON.parse(JSON.stringify(state.memoryBlocks))
  allocatedJobs.value = JSON.parse(JSON.stringify(state.allocatedJobs))
  memorySize.value = state.memorySize
  allocationAlgorithm.value = state.algorithm
  
  addLog(`已恢复到状态: ${state.action}`, 'info')
}

// 监听内存大小变化
watch(memorySize, () => {
  if (!isRunning.value) {
    initializeMemory()
  }
})

// 监听内存块变化，保存历史状态
watch(memoryBlocks, () => {
  if (memoryAllocatorRef.value && !isAutoMode.value) {
    nextTick(() => {
      // 这里可以添加状态保存逻辑
    })
  }
}, { deep: true })

// 初始化
initializeMemory()
</script>

<style scoped>
#app {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  min-height: 100vh;
  color: #333;
}

.header {
  background: rgba(255, 255, 255, 0.98);
  backdrop-filter: blur(20px);
  text-align: center;
  padding: 32px 20px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.12);
  border-bottom: 1px solid rgba(255, 255, 255, 0.2);
  position: relative;
  overflow: hidden;
}

.header::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.05) 0%, rgba(118, 75, 162, 0.05) 100%);
  pointer-events: none;
}

.header h1 {
  margin: 0;
  color: #2c3e50;
  font-size: 2.75rem;
  font-weight: 700;
  letter-spacing: -0.02em;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  background-clip: text;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  position: relative;
  z-index: 1;
}

.header p {
  margin: 8px 0 0 0;
  color: #64748b;
  font-size: 1.125rem;
  font-weight: 500;
  letter-spacing: 0.025em;
  position: relative;
  z-index: 1;
}

.main-container {
  display: flex;
  gap: 24px;
  padding: 24px;
  max-width: 1600px;
  margin: 0 auto;
}

.control-panel {
  background: rgba(255, 255, 255, 0.98);
  backdrop-filter: blur(20px);
  border-radius: 20px;
  padding: 28px;
  width: 380px;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.08), 
              0 0 0 1px rgba(255, 255, 255, 0.5);
  border: 1px solid rgba(255, 255, 255, 0.2);
  height: fit-content;
  position: relative;
  overflow: hidden;
}

.control-panel::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.02) 0%, rgba(118, 75, 162, 0.02) 100%);
  pointer-events: none;
}

.config-section, .job-section {
  margin-bottom: 32px;
  padding-bottom: 24px;
  border-bottom: 1px solid rgba(226, 232, 240, 0.8);
  position: relative;
  z-index: 1;
}

.config-section h3, .job-section h3 {
  margin: 0 0 20px 0;
  color: #1e293b;
  font-size: 1.25rem;
  font-weight: 600;
  letter-spacing: -0.025em;
  display: flex;
  align-items: center;
  gap: 8px;
}

.config-section h3::before {
  content: '⚙️';
  font-size: 1.1rem;
}

.job-section h3::before {
  content: '📋';
  font-size: 1.1rem;
}

.config-item {
  margin-bottom: 20px;
}

.config-item label {
  display: block;
  margin-bottom: 8px;
  font-weight: 600;
  color: #374151;
  font-size: 14px;
  letter-spacing: 0.025em;
}

.config-item input, .config-item select {
  width: 100%;
  padding: 12px 16px;
  border: 2px solid #e2e8f0;
  border-radius: 12px;
  font-size: 14px;
  font-weight: 500;
  background: rgba(255, 255, 255, 0.8);
  backdrop-filter: blur(10px);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}

.config-item input:focus, .config-item select:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1),
              0 4px 12px rgba(102, 126, 234, 0.15);
  transform: translateY(-1px);
}

.config-item input:hover:not(:focus), .config-item select:hover:not(:focus) {
  border-color: #cbd5e1;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

.job-input {
  margin-bottom: 24px;
  position: relative;
  z-index: 1;
}

.input-group {
  margin-bottom: 16px;
}

.input-group label {
  display: block;
  margin-bottom: 8px;
  font-weight: 600;
  color: #374151;
  font-size: 14px;
  letter-spacing: 0.025em;
}

.input-group input {
  width: 100%;
  padding: 12px 16px;
  border: 2px solid #e2e8f0;
  border-radius: 12px;
  font-size: 14px;
  font-weight: 500;
  background: rgba(255, 255, 255, 0.8);
  backdrop-filter: blur(10px);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}

.input-group input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1),
              0 4px 12px rgba(102, 126, 234, 0.15);
  transform: translateY(-1px);
}

.input-group input:hover:not(:focus) {
  border-color: #cbd5e1;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

.job-queue h4, .allocated-jobs h4 {
  margin: 0 0 16px 0;
  color: #1e293b;
  font-size: 1.1rem;
  font-weight: 600;
  letter-spacing: -0.025em;
  display: flex;
  align-items: center;
  gap: 8px;
}

.job-queue h4::before {
  content: '⏳';
  font-size: 1rem;
}

.allocated-jobs h4::before {
  content: '💾';
  font-size: 1rem;
}

.job-list, .allocated-list {
  max-height: 180px;
  overflow-y: auto;
  border: 1px solid rgba(226, 232, 240, 0.8);
  border-radius: 12px;
  padding: 12px;
  background: rgba(248, 250, 252, 0.8);
  backdrop-filter: blur(10px);
}

.job-item, .allocated-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px;
  margin-bottom: 8px;
  background: rgba(255, 255, 255, 0.9);
  border-radius: 8px;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  border: 1px solid rgba(226, 232, 240, 0.5);
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}

.job-item:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.job-item.current {
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.1) 0%, rgba(118, 75, 162, 0.1) 100%);
  border: 2px solid #667eea;
  box-shadow: 0 4px 20px rgba(102, 126, 234, 0.25);
  transform: translateY(-2px);
}

.allocated-item {
  color: white;
  font-weight: 600;
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}

.allocated-item:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.25);
}

.control-buttons {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 24px;
  position: relative;
  z-index: 1;
}

button {
  padding: 14px 20px;
  border: none;
  border-radius: 12px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  box-shadow: 0 4px 15px rgba(102, 126, 234, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.2);
  position: relative;
  overflow: hidden;
}

button::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  transition: left 0.5s;
}

button:hover:not(:disabled)::before {
  left: 100%;
}

button:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(102, 126, 234, 0.5);
}

button:active:not(:disabled) {
  transform: translateY(0);
}

button:disabled {
  background: linear-gradient(135deg, #cbd5e1 0%, #94a3b8 100%);
  cursor: not-allowed;
  transform: none;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.allocated-item button, .job-item button {
  padding: 6px 12px;
  font-size: 12px;
  border-radius: 6px;
  background: rgba(220, 53, 69, 0.9);
  box-shadow: 0 2px 8px rgba(220, 53, 69, 0.3);
}

.allocated-item button:hover:not(:disabled), .job-item button:hover:not(:disabled) {
  background: rgba(220, 53, 69, 1);
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(220, 53, 69, 0.4);
}

.job-input button {
  margin-top: 8px;
  background: linear-gradient(135deg, #10b981 0%, #059669 100%);
  box-shadow: 0 4px 15px rgba(16, 185, 129, 0.4);
}

.job-input button:hover:not(:disabled) {
  box-shadow: 0 8px 25px rgba(16, 185, 129, 0.5);
}

.visualization-area {
  flex: 1;
  background: rgba(255, 255, 255, 0.98);
  backdrop-filter: blur(20px);
  border-radius: 20px;
  padding: 28px;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.08), 
              0 0 0 1px rgba(255, 255, 255, 0.5);
  border: 1px solid rgba(255, 255, 255, 0.2);
  position: relative;
  overflow: hidden;
}

.visualization-area::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.02) 0%, rgba(118, 75, 162, 0.02) 100%);
  pointer-events: none;
}

.memory-info {
  position: relative;
  z-index: 1;
}

.memory-info h3 {
  margin: 0 0 20px 0;
  color: #1e293b;
  font-size: 1.5rem;
  font-weight: 600;
  letter-spacing: -0.025em;
  display: flex;
  align-items: center;
  gap: 8px;
}

.memory-info h3::before {
  content: '🧠';
  font-size: 1.2rem;
}

.stats {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 16px;
  margin-bottom: 24px;
}

.stat-item {
  background: linear-gradient(135deg, rgba(255, 255, 255, 0.9) 0%, rgba(248, 250, 252, 0.9) 100%);
  backdrop-filter: blur(10px);
  padding: 20px;
  border-radius: 16px;
  text-align: center;
  border: 1px solid rgba(226, 232, 240, 0.8);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.05);
  position: relative;
  overflow: hidden;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.stat-item::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 4px;
  height: 100%;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.stat-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.12);
}

.stat-item span {
  font-weight: 600;
  color: #1e293b;
  font-size: 16px;
  letter-spacing: 0.025em;
}

.memory-blocks {
  display: flex;
  border: 2px solid rgba(51, 65, 85, 0.8);
  border-radius: 16px;
  min-height: 450px;
  margin-bottom: 24px;
  overflow-x: auto;
  background: linear-gradient(135deg, rgba(248, 250, 252, 0.8) 0%, rgba(241, 245, 249, 0.8) 100%);
  backdrop-filter: blur(10px);
  position: relative;
  z-index: 1;
  box-shadow: inset 0 2px 8px rgba(0, 0, 0, 0.05);
}

.memory-block {
  min-width: 140px;
  border-right: 1px solid rgba(203, 213, 225, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  backdrop-filter: blur(5px);
}

.memory-block:last-child {
  border-right: none;
}

.memory-block:hover {
  transform: scale(1.02);
  z-index: 2;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15);
}

.memory-block.allocated:hover {
  filter: brightness(1.1);
}

.memory-block.free {
  background: linear-gradient(135deg, 
    rgba(236, 240, 241, 0.9) 0%, 
    rgba(189, 195, 199, 0.9) 25%, 
    rgba(236, 240, 241, 0.9) 50%, 
    rgba(189, 195, 199, 0.9) 75%, 
    rgba(236, 240, 241, 0.9) 100%);
  background-size: 30px 30px;
  position: relative;
}

.memory-block.free::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(45deg, 
    transparent 25%, 
    rgba(148, 163, 184, 0.1) 25%, 
    rgba(148, 163, 184, 0.1) 50%, 
    transparent 50%, 
    transparent 75%, 
    rgba(148, 163, 184, 0.1) 75%);
  background-size: 20px 20px;
  background-position: 0 0, 10px 10px;
}

.memory-block.highlight {
  box-shadow: inset 0 0 0 3px #f59e0b, 0 4px 20px rgba(245, 158, 11, 0.4);
  animation: pulse 2s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% { box-shadow: inset 0 0 0 3px #f59e0b, 0 4px 20px rgba(245, 158, 11, 0.4); }
  50% { box-shadow: inset 0 0 0 3px #f97316, 0 8px 30px rgba(249, 115, 22, 0.6); }
}

.memory-block.allocated {
  border: 1px solid rgba(255, 255, 255, 0.3);
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.2),
              0 2px 8px rgba(0, 0, 0, 0.1);
}

.block-info {
  text-align: center;
  padding: 12px;
  color: white;
  text-shadow: 0 1px 3px rgba(0, 0, 0, 0.4);
  position: relative;
  z-index: 1;
}

.memory-block.free .block-info {
  color: #64748b;
  text-shadow: none;
  font-weight: 600;
}

.block-label {
  font-weight: 700;
  margin-bottom: 6px;
  font-size: 14px;
  letter-spacing: 0.025em;
}

.block-size {
  font-size: 13px;
  margin-bottom: 4px;
  opacity: 0.9;
  font-weight: 600;
}

.block-address {
  font-size: 11px;
  opacity: 0.8;
  font-weight: 500;
  letter-spacing: 0.025em;
}

.execution-log {
  height: 320px;
  position: relative;
  z-index: 1;
}

.execution-log h4 {
  margin: 0 0 16px 0;
  color: #1e293b;
  font-size: 1.25rem;
  font-weight: 600;
  letter-spacing: -0.025em;
  display: flex;
  align-items: center;
  gap: 8px;
}

.execution-log h4::before {
  content: '📜';
  font-size: 1.1rem;
}

.log-content {
  height: 270px;
  overflow-y: auto;
  border: 1px solid rgba(226, 232, 240, 0.8);
  border-radius: 12px;
  padding: 16px;
  background: linear-gradient(135deg, rgba(248, 250, 252, 0.9) 0%, rgba(241, 245, 249, 0.9) 100%);
  backdrop-filter: blur(10px);
  box-shadow: inset 0 2px 8px rgba(0, 0, 0, 0.05);
}

.log-item {
  margin-bottom: 12px;
  padding: 12px 16px;
  border-radius: 8px;
  font-size: 14px;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  border: 1px solid transparent;
  position: relative;
  backdrop-filter: blur(5px);
}

.log-item::before {
  content: '';
  position: absolute;
  left: 0;
  top: 0;
  bottom: 0;
  width: 4px;
  border-radius: 0 4px 4px 0;
}

.log-item:hover {
  transform: translateX(4px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.log-item.success {
  background: linear-gradient(135deg, rgba(34, 197, 94, 0.1) 0%, rgba(21, 128, 61, 0.05) 100%);
  border-color: rgba(34, 197, 94, 0.2);
  color: #15803d;
}

.log-item.success::before {
  background: linear-gradient(135deg, #22c55e 0%, #15803d 100%);
}

.log-item.error {
  background: linear-gradient(135deg, rgba(239, 68, 68, 0.1) 0%, rgba(185, 28, 28, 0.05) 100%);
  border-color: rgba(239, 68, 68, 0.2);
  color: #b91c1c;
}

.log-item.error::before {
  background: linear-gradient(135deg, #ef4444 0%, #b91c1c 100%);
}

.log-item.info {
  background: linear-gradient(135deg, rgba(59, 130, 246, 0.1) 0%, rgba(29, 78, 216, 0.05) 100%);
  border-color: rgba(59, 130, 246, 0.2);
  color: #1d4ed8;
}

.log-item.info::before {
  background: linear-gradient(135deg, #3b82f6 0%, #1d4ed8 100%);
}

.log-time {
  font-weight: 700;
  margin-right: 12px;
  opacity: 0.8;
  font-size: 13px;
}

.log-message {
  font-weight: 500;
  letter-spacing: 0.025em;
}

/* 响应式设计 */
@media (max-width: 1200px) {
  .main-container {
    flex-direction: column;
  }
  
  .control-panel {
    width: 100%;
  }
  
  .stats {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 768px) {
  .stats {
    grid-template-columns: 1fr;
  }
  
  .memory-block {
    min-width: 80px;
  }
  
  .header h1 {
    font-size: 2rem;
  }
}
</style>
