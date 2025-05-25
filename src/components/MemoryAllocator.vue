<template>
  <div class="memory-allocator">
    <!-- 内存碎片化分析 -->
    <div class="analysis-section">
      <h4>内存碎片化分析</h4>
      <div class="analysis-stats">
        <div class="stat-card">
          <div class="stat-value">{{ fragmentationCount }}</div>
          <div class="stat-label">碎片数量</div>
        </div>
        <div class="stat-card">
          <div class="stat-value">{{ largestFreeBlock }}KB</div>
          <div class="stat-label">最大空闲块</div>
        </div>
        <div class="stat-card">
          <div class="stat-value">{{ averageFragmentSize }}KB</div>
          <div class="stat-label">平均碎片大小</div>
        </div>
        <div class="stat-card">
          <div class="stat-value">{{ fragmentationRatio }}%</div>
          <div class="stat-label">碎片化率</div>
        </div>
      </div>
    </div>

    <!-- 算法性能对比 -->
    <div class="performance-section">
      <h4>算法性能对比</h4>
      <div class="performance-chart">
        <div class="chart-container">
          <canvas ref="performanceChart" width="400" height="200"></canvas>
        </div>
        <div class="performance-metrics">
          <div class="metric-item">
            <span class="metric-label">平均分配时间:</span>
            <span class="metric-value">{{ averageAllocationTime }}ms</span>
          </div>
          <div class="metric-item">
            <span class="metric-label">成功分配率:</span>
            <span class="metric-value">{{ allocationSuccessRate }}%</span>
          </div>
          <div class="metric-item">
            <span class="metric-label">内存利用效率:</span>
            <span class="metric-value">{{ memoryEfficiency }}%</span>
          </div>
        </div>
      </div>
    </div>

    <!-- 预设场景 -->
    <div class="scenario-section">
      <h4>预设测试场景</h4>
      <div class="scenario-buttons">
        <button 
          v-for="scenario in testScenarios" 
          :key="scenario.name"
          @click="loadScenario(scenario)"
          class="scenario-btn"
        >
          {{ scenario.name }}
        </button>
      </div>
      <div class="scenario-description" v-if="currentScenario">
        <h5>{{ currentScenario.name }}</h5>
        <p>{{ currentScenario.description }}</p>
        <div class="scenario-jobs">
          <span v-for="job in currentScenario.jobs" :key="job.name" class="job-tag">
            {{ job.name }}({{ job.size }}KB)
          </span>
        </div>
      </div>
    </div>

    <!-- 自动模式控制 -->
    <div class="auto-mode-section">
      <h4>自动模式</h4>
      <div class="auto-controls">
        <div class="speed-control">
          <label>播放速度:</label>
          <input 
            v-model="autoSpeed" 
            type="range" 
            min="100" 
            max="2000" 
            step="100"
            class="speed-slider"
          />
          <span>{{ autoSpeed }}ms</span>
        </div>
        <div class="auto-buttons">
          <button @click="startAutoMode" :disabled="isAutoMode || !canStartAuto">
            自动播放
          </button>
          <button @click="stopAutoMode" :disabled="!isAutoMode">
            停止
          </button>
        </div>
      </div>
    </div>

    <!-- 内存状态历史记录 -->
    <div class="history-section">
      <h4>状态历史</h4>
      <div class="history-timeline">
        <div 
          v-for="(state, index) in memoryHistory" 
          :key="index"
          class="history-item"
          :class="{ 'active': index === currentHistoryIndex }"
          @click="restoreState(index)"
        >
          <div class="history-step">{{ index + 1 }}</div>
          <div class="history-info">
            <div class="history-action">{{ state.action }}</div>
            <div class="history-time">{{ state.timestamp }}</div>
          </div>
        </div>
      </div>
      <div class="history-controls">
        <button @click="previousState" :disabled="currentHistoryIndex <= 0">
          上一步
        </button>
        <button @click="nextState" :disabled="currentHistoryIndex >= memoryHistory.length - 1">
          下一步
        </button>
      </div>
    </div>

    <!-- 导出功能 -->
    <div class="export-section">
      <h4>数据导出</h4>
      <div class="export-buttons">
        <button @click="exportToJSON">导出配置</button>
        <button @click="exportToImage">导出图片</button>
        <button @click="exportReport">生成报告</button>
      </div>
      <div class="import-section">
        <input 
          ref="fileInput" 
          type="file" 
          accept=".json" 
          @change="importConfiguration"
          style="display: none"
        />
        <button @click="$refs.fileInput.click()">导入配置</button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, watch, nextTick } from 'vue'

// Props
interface Props {
  memoryBlocks: any[]
  allocatedJobs: any[]
  memorySize: number
  currentAlgorithm: string
  isRunning: boolean
}

const props = defineProps<Props>()

// Emits
const emit = defineEmits(['load-scenario', 'start-auto', 'stop-auto', 'restore-state'])

// 响应式数据
const autoSpeed = ref(1000)
const isAutoMode = ref(false)
const currentScenario = ref(null)
const currentHistoryIndex = ref(0)
const memoryHistory = ref([])
const performanceChart = ref(null)

// 性能统计
const allocationTimes = ref([])
const allocationResults = ref([])

// 测试场景
const testScenarios = ref([
  {
    name: '小任务密集',
    description: '多个小任务同时申请内存，测试碎片化情况',
    jobs: [
      { name: 'P1', size: 50 },
      { name: 'P2', size: 30 },
      { name: 'P3', size: 40 },
      { name: 'P4', size: 25 },
      { name: 'P5', size: 35 }
    ]
  },
  {
    name: '大任务冲击',
    description: '大任务与小任务混合，测试算法适应性',
    jobs: [
      { name: 'P1', size: 200 },
      { name: 'P2', size: 50 },
      { name: 'P3', size: 300 },
      { name: 'P4', size: 80 },
      { name: 'P5', size: 150 }
    ]
  },
  {
    name: '递增序列',
    description: '任务大小递增，测试最佳/最坏适应算法',
    jobs: [
      { name: 'P1', size: 50 },
      { name: 'P2', size: 100 },
      { name: 'P3', size: 150 },
      { name: 'P4', size: 200 },
      { name: 'P5', size: 250 }
    ]
  },
  {
    name: '内存压力测试',
    description: '接近内存上限的分配测试',
    jobs: [
      { name: 'P1', size: 400 },
      { name: 'P2', size: 300 },
      { name: 'P3', size: 250 },
      { name: 'P4', size: 200 },
      { name: 'P5', size: 150 }
    ]
  }
])

// 计算属性
const fragmentationCount = computed(() => {
  return props.memoryBlocks.filter(block => !block.allocated).length
})

const largestFreeBlock = computed(() => {
  const freeBlocks = props.memoryBlocks.filter(block => !block.allocated)
  return freeBlocks.length > 0 ? Math.max(...freeBlocks.map(block => block.size)) : 0
})

const averageFragmentSize = computed(() => {
  const freeBlocks = props.memoryBlocks.filter(block => !block.allocated)
  if (freeBlocks.length === 0) return 0
  const totalSize = freeBlocks.reduce((sum, block) => sum + block.size, 0)
  return Math.round(totalSize / freeBlocks.length)
})

const fragmentationRatio = computed(() => {
  const freeMemory = props.memoryBlocks
    .filter(block => !block.allocated)
    .reduce((sum, block) => sum + block.size, 0)
  
  if (freeMemory === 0) return 0
  
  const externalFragmentation = freeMemory - largestFreeBlock.value
  return Math.round((externalFragmentation / freeMemory) * 100)
})

const averageAllocationTime = computed(() => {
  if (allocationTimes.value.length === 0) return 0
  const sum = allocationTimes.value.reduce((a, b) => a + b, 0)
  return Math.round(sum / allocationTimes.value.length * 1000) / 1000
})

const allocationSuccessRate = computed(() => {
  if (allocationResults.value.length === 0) return 100
  const successCount = allocationResults.value.filter(result => result).length
  return Math.round((successCount / allocationResults.value.length) * 100)
})

const memoryEfficiency = computed(() => {
  const allocatedMemory = props.memoryBlocks
    .filter(block => block.allocated)
    .reduce((sum, block) => sum + block.size, 0)
  
  return Math.round((allocatedMemory / props.memorySize) * 100)
})

const canStartAuto = computed(() => {
  return !props.isRunning && currentScenario.value && currentScenario.value.jobs.length > 0
})

// 方法
const loadScenario = (scenario) => {
  currentScenario.value = scenario
  emit('load-scenario', scenario)
}

const startAutoMode = () => {
  if (!canStartAuto.value) return
  
  isAutoMode.value = true
  emit('start-auto', {
    scenario: currentScenario.value,
    speed: autoSpeed.value
  })
}

const stopAutoMode = () => {
  isAutoMode.value = false
  emit('stop-auto')
}

const saveCurrentState = (action: string) => {
  const state = {
    action,
    timestamp: new Date().toLocaleTimeString(),
    memoryBlocks: JSON.parse(JSON.stringify(props.memoryBlocks)),
    allocatedJobs: JSON.parse(JSON.stringify(props.allocatedJobs)),
    memorySize: props.memorySize,
    algorithm: props.currentAlgorithm
  }
  
  memoryHistory.value.push(state)
  currentHistoryIndex.value = memoryHistory.value.length - 1
  
  // 限制历史记录数量
  if (memoryHistory.value.length > 20) {
    memoryHistory.value.shift()
    currentHistoryIndex.value--
  }
}

const restoreState = (index: number) => {
  if (index >= 0 && index < memoryHistory.value.length) {
    currentHistoryIndex.value = index
    const state = memoryHistory.value[index]
    emit('restore-state', state)
  }
}

const previousState = () => {
  if (currentHistoryIndex.value > 0) {
    restoreState(currentHistoryIndex.value - 1)
  }
}

const nextState = () => {
  if (currentHistoryIndex.value < memoryHistory.value.length - 1) {
    restoreState(currentHistoryIndex.value + 1)
  }
}

const exportToJSON = () => {
  const data = {
    memorySize: props.memorySize,
    algorithm: props.currentAlgorithm,
    jobs: currentScenario.value?.jobs || [],
    memoryState: props.memoryBlocks,
    allocatedJobs: props.allocatedJobs,
    timestamp: new Date().toISOString()
  }
  
  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `memory-simulation-${Date.now()}.json`
  a.click()
  URL.revokeObjectURL(url)
}

const exportToImage = () => {
  // 这里可以实现将内存可视化导出为图片的功能
  // 使用 html2canvas 或类似库
  console.log('导出图片功能待实现')
}

const exportReport = () => {
  const report = `
内存分配模拟报告
================

时间: ${new Date().toLocaleString()}
算法: ${props.currentAlgorithm}
总内存: ${props.memorySize}KB

性能统计:
- 碎片数量: ${fragmentationCount.value}
- 最大空闲块: ${largestFreeBlock.value}KB
- 碎片化率: ${fragmentationRatio.value}%
- 平均分配时间: ${averageAllocationTime.value}ms
- 分配成功率: ${allocationSuccessRate.value}%
- 内存利用效率: ${memoryEfficiency.value}%

已分配作业:
${props.allocatedJobs.map(job => `- ${job.name}: ${job.size}KB`).join('\n')}

内存块状态:
${props.memoryBlocks.map((block, index) => 
  `块${index + 1}: ${block.allocated ? `已分配(${block.jobName})` : '空闲'} - ${block.size}KB`
).join('\n')}
`
  
  const blob = new Blob([report], { type: 'text/plain;charset=utf-8' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `memory-report-${Date.now()}.txt`
  a.click()
  URL.revokeObjectURL(url)
}

const importConfiguration = (event) => {
  const file = event.target.files[0]
  if (!file) return
  
  const reader = new FileReader()
  reader.onload = (e) => {
    try {
      const data = JSON.parse(e.target.result)
      emit('load-scenario', {
        name: '导入配置',
        description: `导入于 ${new Date(data.timestamp).toLocaleString()}`,
        jobs: data.jobs,
        memorySize: data.memorySize,
        algorithm: data.algorithm
      })
    } catch (error) {
      console.error('导入配置失败:', error)
    }
  }
  reader.readAsText(file)
}

const drawPerformanceChart = () => {
  // 这里可以实现性能图表绘制
  // 使用 Canvas API 或 Chart.js
  if (!performanceChart.value) return
  
  const ctx = performanceChart.value.getContext('2d')
  ctx.clearRect(0, 0, 400, 200)
  
  // 简单的柱状图示例
  const algorithms = ['首次适应', '最佳适应', '最坏适应', '循环首次']
  const values = [85, 92, 78, 88] // 示例数据
  
  algorithms.forEach((alg, index) => {
    const x = index * 90 + 20
    const height = values[index] * 1.5
    const y = 200 - height
    
    ctx.fillStyle = '#3498db'
    ctx.fillRect(x, y, 60, height)
    
    ctx.fillStyle = '#2c3e50'
    ctx.font = '12px Arial'
    ctx.textAlign = 'center'
    ctx.fillText(alg, x + 30, 190)
    ctx.fillText(values[index] + '%', x + 30, y - 5)
  })
}

// 监听器
watch(() => props.memoryBlocks, () => {
  nextTick(() => {
    drawPerformanceChart()
  })
}, { deep: true })

// 生命周期
onMounted(() => {
  drawPerformanceChart()
  
  // 初始化历史记录
  saveCurrentState('系统初始化')
})

// 暴露方法给父组件
defineExpose({
  saveCurrentState,
  startAutoMode,
  stopAutoMode
})
</script>

<style scoped>
.memory-allocator {
  background: rgba(255, 255, 255, 0.98);
  backdrop-filter: blur(20px);
  border-radius: 20px;
  padding: 28px;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.08), 
              0 0 0 1px rgba(255, 255, 255, 0.5);
  border: 1px solid rgba(255, 255, 255, 0.2);
  margin-top: 24px;
  position: relative;
  overflow: hidden;
}

.memory-allocator::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.02) 0%, rgba(118, 75, 162, 0.02) 100%);
  pointer-events: none;
}

.analysis-section,
.performance-section,
.scenario-section,
.auto-mode-section,
.history-section,
.export-section {
  margin-bottom: 32px;
  padding-bottom: 24px;
  border-bottom: 1px solid rgba(226, 232, 240, 0.8);
  position: relative;
  z-index: 1;
}

.analysis-section h4,
.performance-section h4,
.scenario-section h4,
.auto-mode-section h4,
.history-section h4,
.export-section h4 {
  margin: 0 0 20px 0;
  color: #1e293b;
  font-size: 1.25rem;
  font-weight: 600;
  letter-spacing: -0.025em;
  display: flex;
  align-items: center;
  gap: 8px;
}

.analysis-section h4::before {
  content: '📊';
  font-size: 1.1rem;
}

.performance-section h4::before {
  content: '⚡';
  font-size: 1.1rem;
}

.scenario-section h4::before {
  content: '🧪';
  font-size: 1.1rem;
}

.auto-mode-section h4::before {
  content: '🤖';
  font-size: 1.1rem;
}

.history-section h4::before {
  content: '🕒';
  font-size: 1.1rem;
}

.export-section h4::before {
  content: '💾';
  font-size: 1.1rem;
}

.analysis-stats {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
  gap: 16px;
}

.stat-card {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 20px 16px;
  border-radius: 16px;
  text-align: center;
  box-shadow: 0 8px 25px rgba(102, 126, 234, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.2);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
  overflow: hidden;
}

.stat-card::before {
  content: '';
  position: absolute;
  top: -50%;
  left: -50%;
  width: 200%;
  height: 200%;
  background: linear-gradient(45deg, transparent, rgba(255, 255, 255, 0.1), transparent);
  transform: rotate(45deg);
  transition: all 0.5s;
  opacity: 0;
}

.stat-card:hover::before {
  opacity: 1;
  animation: shimmer 1.5s ease-in-out infinite;
}

@keyframes shimmer {
  0% { transform: translateX(-100%) translateY(-100%) rotate(45deg); }
  100% { transform: translateX(100%) translateY(100%) rotate(45deg); }
}

.stat-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 12px 35px rgba(102, 126, 234, 0.5);
}

.stat-value {
  font-size: 1.75rem;
  font-weight: 700;
  margin-bottom: 6px;
  position: relative;
  z-index: 1;
}

.stat-label {
  font-size: 0.875rem;
  opacity: 0.95;
  font-weight: 500;
  letter-spacing: 0.025em;
  position: relative;
  z-index: 1;
}

.performance-chart {
  display: flex;
  gap: 24px;
  align-items: center;
  flex-wrap: wrap;
}

.chart-container {
  border: 1px solid rgba(226, 232, 240, 0.8);
  border-radius: 16px;
  padding: 20px;
  background: linear-gradient(135deg, rgba(255, 255, 255, 0.9) 0%, rgba(248, 250, 252, 0.9) 100%);
  backdrop-filter: blur(10px);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.05);
  min-width: 300px;
}

.performance-metrics {
  flex: 1;
  min-width: 250px;
}

.metric-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
  padding: 16px;
  background: linear-gradient(135deg, rgba(255, 255, 255, 0.9) 0%, rgba(248, 250, 252, 0.9) 100%);
  backdrop-filter: blur(10px);
  border-radius: 12px;
  border: 1px solid rgba(226, 232, 240, 0.8);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.metric-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.1);
}

.metric-label {
  font-weight: 600;
  color: #374151;
  font-size: 14px;
}

.metric-value {
  color: #667eea;
  font-weight: 700;
  font-size: 16px;
  padding: 6px 12px;
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.1) 0%, rgba(118, 75, 162, 0.05) 100%);
  border-radius: 8px;
  border: 1px solid rgba(102, 126, 234, 0.2);
}

.scenario-buttons {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 12px;
  margin-bottom: 20px;
}

.scenario-btn {
  padding: 14px 20px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  font-size: 14px;
  font-weight: 600;
  box-shadow: 0 4px 15px rgba(102, 126, 234, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.2);
  position: relative;
  overflow: hidden;
}

.scenario-btn::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  transition: left 0.5s;
}

.scenario-btn:hover::before {
  left: 100%;
}

.scenario-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(102, 126, 234, 0.5);
}

.scenario-description {
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.05) 0%, rgba(118, 75, 162, 0.02) 100%);
  backdrop-filter: blur(10px);
  padding: 20px;
  border-radius: 16px;
  border: 1px solid rgba(102, 126, 234, 0.2);
  box-shadow: 0 4px 20px rgba(102, 126, 234, 0.1);
  position: relative;
  overflow: hidden;
}

.scenario-description::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 4px;
  height: 100%;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.scenario-description h5 {
  margin: 0 0 12px 0;
  color: #1e293b;
  font-size: 1.1rem;
  font-weight: 600;
  letter-spacing: -0.025em;
}

.scenario-description p {
  margin: 0 0 16px 0;
  color: #64748b;
  font-size: 14px;
  line-height: 1.6;
  font-weight: 500;
}

.scenario-jobs {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.job-tag {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 6px 12px;
  border-radius: 8px;
  font-size: 12px;
  font-weight: 600;
  box-shadow: 0 2px 8px rgba(102, 126, 234, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.2);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.job-tag:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.auto-controls {
  display: flex;
  flex-direction: column;
  gap: 20px;
  position: relative;
  z-index: 1;
}

.speed-control {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px;
  background: linear-gradient(135deg, rgba(255, 255, 255, 0.9) 0%, rgba(248, 250, 252, 0.9) 100%);
  backdrop-filter: blur(10px);
  border-radius: 12px;
  border: 1px solid rgba(226, 232, 240, 0.8);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.speed-control label {
  font-weight: 600;
  color: #374151;
  min-width: 80px;
}

.speed-slider {
  flex: 1;
  max-width: 250px;
  height: 6px;
  border-radius: 3px;
  background: linear-gradient(135deg, #e2e8f0 0%, #cbd5e1 100%);
  outline: none;
  appearance: none;
}

.speed-slider::-webkit-slider-thumb {
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  cursor: pointer;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.speed-slider::-webkit-slider-thumb:hover {
  transform: scale(1.1);
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.6);
}

.speed-control span {
  font-weight: 600;
  color: #667eea;
  min-width: 60px;
  text-align: right;
}

.auto-buttons {
  display: flex;
  gap: 12px;
}

.auto-buttons button {
  padding: 14px 20px;
  border: none;
  border-radius: 12px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
  overflow: hidden;
}

.auto-buttons button:first-child {
  background: linear-gradient(135deg, #10b981 0%, #059669 100%);
  color: white;
  box-shadow: 0 4px 15px rgba(16, 185, 129, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.auto-buttons button:first-child:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(16, 185, 129, 0.5);
}

.auto-buttons button:last-child {
  background: linear-gradient(135deg, #ef4444 0%, #dc2626 100%);
  color: white;
  box-shadow: 0 4px 15px rgba(239, 68, 68, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.auto-buttons button:last-child:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(239, 68, 68, 0.5);
}

.auto-buttons button:disabled {
  background: linear-gradient(135deg, #cbd5e1 0%, #94a3b8 100%);
  cursor: not-allowed;
  transform: none;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.history-timeline {
  max-height: 240px;
  overflow-y: auto;
  border: 1px solid rgba(226, 232, 240, 0.8);
  border-radius: 12px;
  padding: 16px;
  background: linear-gradient(135deg, rgba(248, 250, 252, 0.9) 0%, rgba(241, 245, 249, 0.9) 100%);
  backdrop-filter: blur(10px);
  box-shadow: inset 0 2px 8px rgba(0, 0, 0, 0.05);
}

.history-item {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 12px;
  margin-bottom: 8px;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  border: 1px solid transparent;
  position: relative;
}

.history-item:hover {
  background: rgba(255, 255, 255, 0.8);
  transform: translateX(4px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.history-item.active {
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.1) 0%, rgba(118, 75, 162, 0.05) 100%);
  border: 1px solid rgba(102, 126, 234, 0.3);
  box-shadow: 0 4px 20px rgba(102, 126, 234, 0.15);
  transform: translateX(8px);
}

.history-step {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 13px;
  font-weight: 700;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.3);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.history-item.active .history-step {
  background: linear-gradient(135deg, #10b981 0%, #059669 100%);
  box-shadow: 0 6px 20px rgba(16, 185, 129, 0.4);
  transform: scale(1.1);
}

.history-info {
  flex: 1;
}

.history-action {
  font-weight: 600;
  margin-bottom: 4px;
  color: #1e293b;
  font-size: 14px;
}

.history-time {
  font-size: 12px;
  color: #64748b;
  font-weight: 500;
}

.history-controls {
  display: flex;
  gap: 12px;
  margin-top: 16px;
}

.history-controls button {
  padding: 10px 16px;
  border: none;
  border-radius: 8px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  box-shadow: 0 2px 8px rgba(102, 126, 234, 0.3);
}

.history-controls button:hover:not(:disabled) {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.history-controls button:disabled {
  background: linear-gradient(135deg, #cbd5e1 0%, #94a3b8 100%);
  cursor: not-allowed;
  transform: none;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);
}

.export-buttons,
.import-section {
  display: flex;
  gap: 12px;
  margin-bottom: 16px;
  flex-wrap: wrap;
}

.export-buttons button,
.import-section button {
  padding: 12px 20px;
  border: none;
  border-radius: 12px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
  overflow: hidden;
}

.export-buttons button {
  background: linear-gradient(135deg, #8b5cf6 0%, #7c3aed 100%);
  color: white;
  box-shadow: 0 4px 15px rgba(139, 92, 246, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.export-buttons button:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(139, 92, 246, 0.5);
}

.import-section button {
  background: linear-gradient(135deg, #f59e0b 0%, #d97706 100%);
  color: white;
  box-shadow: 0 4px 15px rgba(245, 158, 11, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.import-section button:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(245, 158, 11, 0.5);
}

/* 响应式设计 */
@media (max-width: 768px) {
  .performance-chart {
    flex-direction: column;
  }
  
  .analysis-stats {
    grid-template-columns: repeat(2, 1fr);
  }
  
  .scenario-buttons {
    grid-template-columns: 1fr;
  }
}
</style>
