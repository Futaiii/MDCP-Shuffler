<template>
  <div class="app-container">
    <!-- 顶部工具栏 (无变化) -->
    <header class="header">
      <h1 class="title">
        <span class="icon">🎵</span>
        Rhythm Game Picker
      </h1>
      <div class="header-actions">
        <button @click="openAddModal" class="btn btn-primary">
          <span>➕</span> 添加歌曲
        </button>
        <button @click="openRandomModal" class="btn btn-success">
          <span>🎲</span> 随机抽取
        </button>
      </div>
    </header>

    <!-- 筛选栏 (无变化) -->
    <div class="filter-bar">
      <div class="filter-group">
        <label>排序:</label>
        <select v-model="sortType" @change="loadSongs" class="select">
          <option value="title">A-Z</option>
          <option value="level">难度等级</option>
        </select>
      </div>
      <div class="filter-group">
        <label>
          <input type="checkbox" v-model="showFavoritesOnly" @change="loadSongs">
          只显示收藏
        </label>
      </div>
    </div>

    <!-- 歌曲网格 -->
    <main class="main-content">
      <RecycleScroller
          class="scroller songs-grid"
          :items="filteredSongs"
          :item-size="250"
          key-field="id"
          v-slot="{ item }"
      >
        <div
            class="song-card"
            :style="{ backgroundColor: item.color }"
        >
          <div class="card-overlay">
            <button @click="toggleFavorite(item.id)" class="favorite-btn" :class="{ active: item.isFavorite }">
              {{ item.isFavorite ? '⭐' : '☆' }}
            </button>
            <div class="card-content">
              <h3 class="song-title">{{ item.title }}</h3>
              <p class="song-artist">{{ item.artist }}</p>
              <div class="song-meta">
                <span class="level-badge">Lv.{{ item.level || 'N/A' }}</span>
              </div>
            </div>
            <div class="card-actions">
              <button @click="editSong(item)" class="btn-icon" title="编辑">✏️</button>
              <button @click="deleteSong(item)" class="btn-icon" title="删除">🗑️</button>
            </div>
          </div>
        </div>
      </RecycleScroller>

      <div v-if="filteredSongs.length === 0" class="empty-state">
        <div class="empty-icon">🎵</div>
        <p>暂无歌曲，点击"添加歌曲"开始吧！</p>
      </div>
    </main>

    <!-- 添加/编辑歌曲模态框 (无变化) -->
    <transition name="modal">
      <div v-if="showAddModal" class="modal-overlay" @click.self="closeAddModal">
        <div class="modal-content">
          <h2>{{ editingItem ? '编辑歌曲' : '添加歌曲' }}</h2>
          <form @submit.prevent="saveSong" class="form">
            <div class="form-group">
              <label>歌曲名称 *</label>
              <input v-model="formData.title" type="text" required class="input">
            </div>
            <div class="form-group">
              <label>曲包</label>
              <input v-model="formData.artist" type="text" class="input">
            </div>
            <div class="form-group">
              <label>等级</label>
              <input v-model.number="formData.level" type="number" min="0" max="13" step="0.1" placeholder="等级" class="input">
            </div>
            <div class="form-group">
              <label>背景颜色</label>
              <div class="color-picker">
                <div
                    v-for="color in macaronColors"
                    :key="color"
                    class="color-swatch"
                    :style="{ backgroundColor: color }"
                    :class="{ selected: formData.color === color }"
                    @click="formData.color = color"
                ></div>
              </div>
            </div>
            <div class="form-actions">
              <button type="button" @click="closeAddModal" class="btn btn-secondary">取消</button>
              <button type="submit" class="btn btn-primary">保存</button>
            </div>
          </form>
        </div>
      </div>
    </transition>

    <!-- 随机抽取模态框 (无变化) -->
    <transition name="modal">
      <div v-if="showRandomModal" class="modal-overlay" @click.self="showRandomModal = false">
        <div class="modal-content">
          <h2>🎲 随机抽取设置</h2>
          <form @submit.prevent="performRandomPick" class="form">
            <div class="form-group">
              <label>抽取数量</label>
              <div class="stepper-input">
                <button type="button" @click="changeRandomCount(-1)" class="stepper-btn">-</button>
                <input v-model.number="randomOptions.count"  min="1" max="10" required class="input">
                <button type="button" @click="changeRandomCount(1)" class="stepper-btn">+</button>
              </div>
            </div>
            <div class="form-group">
              <label>等级范围</label>
              <div class="range-group">
                <input v-model.number="randomOptions.minLevel" type="number" min="0" step="0.1" placeholder="最低" class="input">
                <span>-</span>
                <input v-model.number="randomOptions.maxLevel" type="number" max="20" step="0.1" placeholder="最高" class="input">
              </div>
            </div>
            <div class="form-group">
              <label class="checkbox-label">
                <input type="checkbox" v-model="randomOptions.onlyFavorites">
                <span>仅从收藏中抽取</span>
              </label>
            </div>
            <div class="form-actions">
              <button type="button" @click="showRandomModal = false" class="btn btn-secondary">取消</button>
              <button type="submit" class="btn btn-success">开始抽取</button>
            </div>
          </form>
        </div>
      </div>
    </transition>

    <!-- 抽取结果模态框 (无变化) -->
    <transition name="modal">
      <div v-if="showResultModal" class="modal-overlay" @click.self="showResultModal = false">
        <div class="modal-content result-modal">
          <h2>🎊 抽取结果</h2>
          <div class="result-grid">
            <div
                v-for="(song, index) in randomResults"
                :key="song.id"
                class="result-card"
                :style="{ backgroundColor: song.color, animationDelay: `${index * 0.1}s` }"
            >
              <div class="result-number">{{ index + 1 }}</div>
              <h3>{{ song.title }}</h3>
              <p>{{ song.artist }}</p>
              <div class="result-meta">
                <span class="level-badge">Lv.{{ song.level }}</span>
              </div>
            </div>
          </div>
          <button @click="showResultModal = false" class="btn btn-primary btn-full">关闭</button>
        </div>
      </div>
    </transition>

    <!-- [新增] 删除确认模态框 -->
    <transition name="modal">
      <div v-if="showDeleteConfirmModal" class="modal-overlay" @click.self="closeDeleteConfirmModal">
        <div class="modal-content">
          <h2>确认删除</h2>
          <p>您确定要删除歌曲 "{{ songToDeleteTitle }}" 吗？此操作无法撤销。</p>
          <div class="form-actions">
            <button type="button" @click="closeDeleteConfirmModal" class="btn btn-secondary">取消</button>
            <button type="button" @click="confirmDelete" class="btn btn-danger">确认删除</button>
          </div>
        </div>
      </div>
    </transition>

  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { RecycleScroller } from 'vue-virtual-scroller'

import { GetAllSongs, AddSong, UpdateSong, DeleteSong as BackendDeleteSong, ToggleFavorite, RandomPick } from '../wailsjs/go/main/App'

const songs = ref([])
const sortType = ref('title')
const showFavoritesOnly = ref(false)
const showAddModal = ref(false)
const showRandomModal = ref(false)
const showResultModal = ref(false)
const editingItem = ref(null)
const randomResults = ref([])

// [新增] 删除确认模态框的状态
const showDeleteConfirmModal = ref(false)
const songToDeleteId = ref(null)
const songToDeleteTitle = ref('')


const macaronColors = [
  '#FFB3BA', '#FFDFBA', '#FFFFBA', '#BAFFC9', '#BAE1FF',
  '#E0BBE4', '#FFB7D5', '#C7CEEA', '#B4E7F5', '#FED9B7',
  '#A0E7E5', '#F7C6C7', '#C9E4DE', '#FFD1DC', '#E4C1F9',
];

const formData = ref({
  title: '',
  artist: '',
  level: 10,
  color: macaronColors[0],
  isFavorite: false
})

const randomOptions = ref({
  count: 1,
  minLevel: 0,
  maxLevel: 0,
  onlyFavorites: false
})

const filteredSongs = computed(() => {
  let result = [...songs.value]

  if (showFavoritesOnly.value) {
    result = result.filter(s => s.isFavorite)
  }

  if (sortType.value === 'title') {
    result.sort((a, b) => a.title.localeCompare(b.title))
  } else if (sortType.value === 'level') {
    result.sort((a, b) => b.level - a.level)
  }

  return result
})

onMounted(() => {
  loadSongs()
})

async function loadSongs() {
  try {
    songs.value = await GetAllSongs()
    console.log(songs.value)
  } catch (err) {
    console.error('加载歌曲失败:', err)
  }
}

function openAddModal() {
  resetForm()
  showAddModal.value = true
}

function editSong(song) {
  editingItem.value = { ...song }

  formData.value = {
    id: song.id,
    title: song.title,
    artist: song.artist,
    level: song.level || 1,
    color: song.color || macaronColors[0],
    isFavorite: song.isFavorite,
  }

  showAddModal.value = true
}

async function saveSong() {
  const songToSave = { ...formData.value }

  try {
    if (editingItem.value) {
      await UpdateSong(songToSave)
    } else {
      await AddSong(songToSave)
    }
    await loadSongs()
    closeAddModal()
  } catch (err) {
    console.error('保存失败:', err)
  }
}

// [修改] deleteSong 函数现在只打开模态框
function deleteSong(song) {
  songToDeleteId.value = song.id;
  songToDeleteTitle.value = song.title;
  showDeleteConfirmModal.value = true;
}

// [新增] 确认删除的逻辑
async function confirmDelete() {
  if (songToDeleteId.value === null) return;
  try {
    // 使用别名 BackendDeleteSong 以避免与本地函数名冲突
    await BackendDeleteSong(songToDeleteId.value)

    await loadSongs()
  } catch (err) {
    console.error('删除失败:', err)
  } finally {
    closeDeleteConfirmModal();
  }
}

// [新增] 关闭删除模态框的函数
function closeDeleteConfirmModal() {
  showDeleteConfirmModal.value = false;
  songToDeleteId.value = null;
  songToDeleteTitle.value = '';
}


async function toggleFavorite(id) {
  try {
    await ToggleFavorite(id)
    await loadSongs()
  } catch (err) {
    console.error('操作失败:', err)
  }
}

async function performRandomPick() {
  try {
    randomResults.value = await RandomPick(randomOptions.value)
    showRandomModal.value = false
    showResultModal.value = true
  } catch (err)
  {
    console.error('抽取失败:', err)
  }
}

function closeAddModal() {
  showAddModal.value = false
  editingItem.value = null
  resetForm()
}

function resetForm() {
  formData.value = {
    title: '',
    artist: '',
    level: 1,
    color: macaronColors[0],
    isFavorite: false
  }
}

function openRandomModal() {
  randomOptions.value.onlyFavorites = showFavoritesOnly.value
  showRandomModal.value = true
}

function changeRandomCount(delta) {
  const current = randomOptions.value.count;
  const newValue = current + delta;
  if (newValue >= 1 && newValue <= 10) {
    randomOptions.value.count = newValue;
  }
}
</script>

<style scoped>
.app-container {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 30px;
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.2);
}

.title {
  font-size: 28px;
  color: white;
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 0;
}

.icon {
  font-size: 32px;
  animation: bounce 2s infinite;
}

@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-10px); }
}

.header-actions {
  display: flex;
  gap: 15px;
}

.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s;
  display: flex;
  align-items: center;
  gap: 8px;
}

.btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
}

.btn-primary {
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
}

.btn-success {
  background: linear-gradient(135deg, #f093fb, #f5576c);
  color: white;
}

.btn-secondary {
  background: rgba(255, 255, 255, 0.2);
  color: white;
}

.filter-bar {
  display: flex;
  gap: 20px;
  padding: 15px 30px;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(10px);
  flex-wrap: wrap;
  align-items: center;
}

.filter-group {
  display: flex;
  align-items: center;
  gap: 10px;
  color: white;
}

.select {
  padding: 8px 12px;
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 6px;
  background: rgba(255, 255, 255, 0.1);
  color: white;
  cursor: pointer;
}

.checkbox-group {
  display: flex;
  gap: 15px;
}

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 5px;
  cursor: pointer;
}

.main-content {
  flex: 1;
  overflow-y: auto;
  padding: 30px;
}

.songs-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 20px;
}

.song-card {
  position: relative;
  height: 200px;
  border-radius: 15px;
  overflow: hidden;
  cursor: pointer;
  transition: all 0.3s;
  background-size: cover;
  background-position: center;
}

.song-card:hover {
  transform: translateY(-8px);
  box-shadow: 0 15px 40px rgba(0, 0, 0, 0.4);
}

.card-overlay {
  position: absolute;
  inset: 0;
  background: linear-gradient(to bottom, rgba(0, 0, 0, 0.3), rgba(0, 0, 0, 0.7));
  padding: 15px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.favorite-btn {
  position: absolute;
  top: 10px;
  right: 10px;
  background: rgba(255, 255, 255, 0.2);
  border: none;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  font-size: 24px;
  cursor: pointer;
  transition: all 0.3s;
  backdrop-filter: blur(5px);
}

.favorite-btn:hover {
  transform: scale(1.2);
}

.favorite-btn.active {
  background: rgba(255, 215, 0, 0.8);
  animation: pulse 0.5s;
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.2); }
}

.card-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  color: white;
}

.song-title {
  font-size: 20px;
  font-weight: bold;
  margin: 0 0 5px 0;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
}

.song-artist {
  font-size: 14px;
  opacity: 0.9;
  margin: 0 0 10px 0;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
}

.song-meta {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.difficulty-badge, .level-badge {
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: bold;
  backdrop-filter: blur(5px);
}

.difficulty-badge {
  background: rgba(255, 255, 255, 0.3);
}

.diff-easy { background: rgba(102, 187, 106, 0.8); }
.diff-normal { background: rgba(66, 165, 245, 0.8); }
.diff-hard { background: rgba(255, 167, 38, 0.8); }
.diff-expert { background: rgba(239, 83, 80, 0.8); }
.diff-master { background: rgba(156, 39, 176, 0.8); }

.level-badge {
  background: rgba(0, 0, 0, 0.5);
  color: white;
}

.card-actions {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
}

.btn-icon {
  background: rgba(255, 255, 255, 0.2);
  border: none;
  border-radius: 8px;
  padding: 8px 12px;
  font-size: 16px;
  cursor: pointer;
  transition: all 0.3s;
  backdrop-filter: blur(5px);
}

.btn-icon:hover {
  background: rgba(255, 255, 255, 0.4);
  transform: scale(1.1);
}

.empty-state {
  text-align: center;
  padding: 60px 20px;
  color: white;
}

.empty-icon {
  font-size: 80px;
  margin-bottom: 20px;
  animation: float 3s ease-in-out infinite;
}

@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-20px); }
}

.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  backdrop-filter: blur(5px);
}

.modal-content {
  background: white;
  border-radius: 20px;
  padding: 30px;
  max-width: 500px;
  width: 90%;
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.5);
}

.modal-content h2 {
  margin: 0 0 20px 0;
  color: #333;
}

.form {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.form-group label {
  font-weight: 600;
  color: #555;
}

.input, .select {
  padding: 10px;
  border: 2px solid #ddd;
  border-radius: 8px;
  font-size: 14px;
  transition: border-color 0.3s;
}

.input:focus, .select:focus {
  outline: none;
  border-color: #667eea;
}

.input-color {
  height: 50px;
  cursor: pointer;
}

.range-group {
  display: flex;
  align-items: center;
  gap: 10px;
}

.range-group .input {
  flex: 1;
}

.form-actions {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
  margin-top: 10px;
}

.btn-full {
  width: 100%;
  margin-top: 20px;
}

.result-modal {
  max-width: 800px;
}

.result-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 15px;
  margin: 20px 0;
}

.result-card {
  position: relative;
  padding: 20px;
  border-radius: 15px;
  text-align: center;
  animation: slideIn 0.5s ease-out;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.2);
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(30px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.result-number {
  position: absolute;
  top: -10px;
  right: -10px;
  width: 40px;
  height: 40px;
  background: linear-gradient(135deg, #f093fb, #f5576c);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  color: white;
  font-size: 18px;
  box-shadow: 0 3px 10px rgba(0, 0, 0, 0.3);
}

.result-card h3 {
  margin: 10px 0 5px 0;
  color: #333;
}

.result-card p {
  margin: 0 0 10px 0;
  color: #666;
  font-size: 14px;
}

.result-meta {
  display: flex;
  gap: 5px;
  justify-content: center;
  flex-wrap: wrap;
}

.modal-enter-active, .modal-leave-active {
  transition: opacity 0.3s;
}

.modal-enter-from, .modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-content,
.modal-leave-active .modal-content {
  transition: transform 0.3s;
}

.modal-enter-from .modal-content,
.modal-leave-to .modal-content {
  transform: scale(0.9);
}


/* 新增或修改的样式 */
.difficulty-level-group {
  display: flex;
  gap: 10px;
  align-items: center;
  margin-bottom: 10px;
}
.difficulty-level-group .select {
  flex: 2;
  background-color: white; /* 确保在模态框中可见 */
  color: #333;
}
.difficulty-level-group .input {
  flex: 1;
}
.btn-icon-small {
  background: none;
  border: none;
  font-size: 18px;
  cursor: pointer;
  padding: 5px;
}
.btn-link {
  background: none;
  border: none;
  color: #667eea;
  cursor: pointer;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 5px;
}

.color-picker {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}
.color-swatch {
  width: 30px;
  height: 30px;
  border-radius: 50%;
  cursor: pointer;
  border: 2px solid transparent;
  transition: all 0.2s;
}
.color-swatch:hover {
  transform: scale(1.1);
}
.color-swatch.selected {
  border-color: #667eea;
  box-shadow: 0 0 0 2px white, 0 0 0 4px #667eea;
}

.stepper-input {
  display: flex;
  align-items: center;
}
.stepper-input .input {
  text-align: center;
  border-left: none;
  border-right: none;
  border-radius: 0;
}
.stepper-btn {
  width: 40px;
  height: 40px;
  border: 2px solid #ddd;
  background-color: #f5f5f5;
  font-size: 20px;
  font-weight: bold;
  cursor: pointer;
}
.stepper-btn:first-child {
  border-radius: 8px 0 0 8px;
}
.stepper-btn:last-child {
  border-radius: 0 8px 8px 0;
}

.btn-danger {
  background-color: #e74c3c;
  color: white;
  border: 1px solid #c0392b;
}

.btn-danger:hover {
  background-color: #c0392b;
}

.scroller {
  /*
    这是最关键的一步！
    虚拟滚动容器必须有一个明确的高度。
    这里我们让它填满 main-content 的剩余空间。
    你需要根据你的布局来调整这个值。
  */
  height: 100%;
  overflow-y: auto;
}

/*
  为了让 .scroller 的 height: 100% 生效，
  它的父容器 .main-content 也需要有明确的高度。
  我们可以使用 flex 布局来实现。
*/
.app-container {
  display: flex;
  flex-direction: column;
  height: 100vh; /* 让整个应用占满视口高度 */
}

.main-content {
  flex-grow: 1; /* 让主内容区填满剩余空间 */
  overflow: hidden; /* 防止内部滚动条影响布局 */
  position: relative; /* 为 empty-state 定位 */
}

/*
  vue-virtual-scroller 会生成一些自己的内部元素，
  我们需要确保 songs-grid 的样式能正确应用。
  把 grid 样式应用到 .scroller 本身通常是最简单的。
*/
.songs-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 20px;
  padding: 20px;
}
</style>