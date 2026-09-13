<script setup>
import { ref, defineProps } from 'vue'
const emit = defineEmits(['search', 'mode-change', 'length'])
const searchText = ref('')
const mode = ref('search')
const selectDrawType = ref('rect')
const isDrawing = ref(false)
const props = defineProps({
  returnLength: {
    type: Number,
    default: 0
  }
})
function setMode(m) {
  mode.value = m
  isDrawing.value = false
}

function doSearch() {
  emit('search', searchText.value)
}

function doLength() {
  emit('length', { start: true })
}

function handleDrawBtn() {
  if (!isDrawing.value) {
    // 进入绘制模式
    emit('mode-change', 'select', { run: true, drawType: selectDrawType.value })
    isDrawing.value = true
  } else {
    // 点击完成，结束绘制
    emit('mode-change', 'select', { finish: true })
    isDrawing.value = false
  }
}

function doClear() {
  emit('mode-change', 'select', { clear: true })
  isDrawing.value = false // 清除后重置按钮文字
}

function doLengthClear() {
  emit('length', { clear: true })
}
</script>

<template>
  <div class="bar-container">
    <!-- 功能tab 栏 -->
    <div class="tab-bar">
      <div class="tab-item" :class="{ active: mode === 'search' }" @click="setMode('search')">
        <span>搜索</span>
      </div>
      <div class="tab-item" :class="{ active: mode === 'select' }" @click="setMode('select')">
        <span>框选查询</span>
      </div>
      <div class="tab-item" :class="{ active: mode === 'length' }" @click="setMode('length')">
        <span>距离</span>
      </div>
    </div>
    <!-- 搜索功能区 -->
    <div class="content-panel" v-if="mode === 'search'">
      <div class="search-wrapper">
        <input type="text" class="search-input" v-model="searchText" placeholder="输入地名，如：安徽大学" @keyup.enter="doSearch">
        <button class="search-button" @click="doSearch"><span>搜索</span></button>
      </div>
    </div>

    <!-- 框选查询内容区 -->
    <div class="content-panel" v-else-if="mode === 'select'">
      <div class="select-wrapper">
        <span class="select-tip">在地图上点击两点绘制矩形（可绘制多个），点击完成即可查询框内要素</span>
        <!-- ==========新增：多选一按钮组========== -->
        <div class="draw-type-group">
          <label>
            <input type="radio" v-model="selectDrawType" value="rect">
            矩形边框
          </label>
          <label>
            <input type="radio" v-model="selectDrawType" value="custom">
            自定义边框
          </label>
        </div>
        <div class="select-actions">
          <button class="btn primary" @click="handleDrawBtn">
            <span>{{ isDrawing ? '完成' : '开始绘制ROI' }}</span>
          </button>
          <button class="btn ghost" @click="doClear">
            <span>清除</span>
          </button>
        </div>
      </div>
    </div>

    <div class="content-panel" v-else>
      <div class="search-wrapper">
        <span class="select-tip">在地图上绘制直线，点击查询即可获得路径的距离</span>
        <span>距离为{{ returnLength }}km</span>
        <button class="search-button" @click="doLength"><span>绘制</span></button>
        <button class="search-button" @click="doLengthClear"><span>取消</span></button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.bar-container {
  display: flex;
  flex-direction: column;
  border-radius: 14px;
  overflow: hidden;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.28);
  backdrop-filter: blur(12px);
  background: rgba(255, 255, 255, 0.92);
  border: 1px solid rgba(255, 255, 255, 0.6);
}

/* Tab 栏 */
.tab-bar {
  display: flex;
  align-items: center;
  padding: 0 4px;
  background: linear-gradient(135deg, #1a73e8 0%, #0d47a1 100%);
  min-height: 40px;
}

.tab-item {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 16px;
  color: rgba(255, 255, 255, 0.7);
  font-size: 13px;
  font-weight: 500;
  cursor: default;
  border-radius: 8px 8px 0 0;
  transition: all 0.2s;
}

.tab-item.active {
  color: #1a73e8;
  background: rgba(255, 255, 255, 0.92);
}

.tab-icon {
  width: 16px;
  height: 16px;
}

/* 内容面板 */
.content-panel {
  padding: 14px 16px;
  background: rgba(255, 255, 255, 0.92);
}

.search-wrapper {
  display: flex;
  align-items: center;
  gap: 8px;
}

.search-icon {
  width: 20px;
  height: 20px;
  color: #5f6368;
  flex-shrink: 0;
}

.search-input {
  flex: 1;
  height: 38px;
  padding: 0 14px;
  border: 1.5px solid #e0e0e0;
  border-radius: 10px;
  font-size: 14px;
  color: #202124;
  background: #f8f9fa;
  outline: none;
  transition: border-color 0.2s, box-shadow 0.2s, background 0.2s;
}

.search-input::placeholder {
  color: #9aa0a6;
}

.search-input:hover {
  border-color: #c4c4c4;
  background: #fff;
}

.search-input:focus {
  border-color: #1a73e8;
  box-shadow: 0 0 0 3px rgba(26, 115, 232, 0.12);
  background: #fff;
}

.search-button {
  display: flex;
  align-items: center;
  gap: 4px;
  height: 38px;
  padding: 0 18px;
  border: none;
  border-radius: 10px;
  background: linear-gradient(135deg, #1a73e8 0%, #1765cc 100%);
  color: #fff;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  white-space: nowrap;
  transition: all 0.2s;
  box-shadow: 0 2px 8px rgba(26, 115, 232, 0.3);
}

.search-button:hover {
  background: linear-gradient(135deg, #1765cc 0%, #1457b8 100%);
  box-shadow: 0 4px 14px rgba(26, 115, 232, 0.45);
  transform: translateY(-1px);
}

.search-button:active {
  transform: translateY(0);
  box-shadow: 0 2px 6px rgba(26, 115, 232, 0.3);
}

.btn-icon {
  width: 16px;
  height: 16px;
}

/* 框选区 */
.select-wrapper {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.select-tip {
  font-size: 12.5px;
  color: #5f6368;
  line-height: 1.5;
}

.select-actions {
  display: flex;
  gap: 8px;
}

.btn {
  display: flex;
  align-items: center;
  gap: 4px;
  height: 34px;
  padding: 0 16px;
  border: none;
  border-radius: 10px;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.btn.primary {
  background: linear-gradient(135deg, #1a73e8, #1765cc);
  color: #fff;
  box-shadow: 0 2px 8px rgba(26, 115, 232, 0.3);
}

.btn.primary:hover {
  background: linear-gradient(135deg, #1765cc, #1457b8);
  box-shadow: 0 4px 14px rgba(26, 115, 232, 0.45);
}

.btn.ghost {
  background: #fff;
  color: #1a73e8;
  border: 1.5px solid #1a73e8;
}

.btn.ghost:hover {
  background: #f0f6ff;
}
</style>
