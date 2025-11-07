<template>
  <div class="app-container">
    <!-- 顶部工具栏 -->
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

    <!-- 筛选栏 -->
    <div class="filter-bar">
      <div class="filter-group search-group">
        <span class="search-icon">🔍</span>
        <input type="text" v-model="searchTerm" placeholder="按歌名或曲包搜索..." class="search-input">
      </div>
      <div class="filter-group">
        <label>排序:</label>
        <select v-model="sortType" class="select">
          <option value="title_asc">歌名 A-Z</option>
          <option value="title_desc">歌名 Z-A</option>
          <option value="level_desc">难度 逆序</option>
          <option value="level_asc">难度 正序</option>
        </select>
      </div>
      <div class="filter-group">
        <label class="checkbox-label">
          <input type="checkbox" v-model="showFavoritesOnly">
          只显示收藏
        </label>
      </div>
    </div>

    <!-- 歌曲网格 -->
    <main class="main-content">
      <transition-group name="card-list" tag="div" class="songs-grid">
        <div
            v-for="song in visibleSongs"
            :key="song.id"
            class="song-card"
            :style="{ backgroundColor: song.color }"
        >
          <div class="card-overlay">
            <button @click="toggleFavorite(song.id)" class="favorite-btn" :class="{ active: song.isFavorite }">
              {{ song.isFavorite ? '⭐' : '☆' }}
            </button>
            <div class="card-content">
              <h3 class="song-title">{{ song.title }}</h3>
              <p class="song-artist">{{ song.artist }}</p>
              <div class="song-meta">
                <span class="level-badge">Lv.{{ song.level || 'N/A' }}</span>
              </div>
            </div>
            <div class="card-actions">
              <!-- 功能1: 新增黑名单切换按钮 -->
              <button @click="toggleBlacklist(song.id)" class="btn-icon blacklist-btn" :class="{ active: song.isBlacklisted }" title="加入黑名单 (随机时不出现)">🚫</button>
              <button @click="editSong(song)" class="btn-icon" title="编辑">✏️</button>
              <button @click="deleteSong(song)" class="btn-icon" title="删除">🗑️</button>
            </div>
          </div>
        </div>
      </transition-group>

      <div ref="loadMoreTrigger" v-show="hasMoreSongs" class="load-more-trigger"></div>

      <div v-if="visibleSongs.length === 0 && !loading" class="empty-state">
        <div class="empty-icon">🎵</div>
        <p>没有找到匹配的歌曲哦！</p>
      </div>
    </main>

    <!-- 添加/编辑歌曲模态框 -->
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
            <!-- 功能1: 新增黑名单复选框 -->
            <div class="form-group">
              <label class="checkbox-label">
                <input type="checkbox" v-model="formData.isBlacklisted">
                加入黑名单 (随机时不出现)
              </label>
            </div>
            <div class="form-actions">
              <button type="button" @click="closeAddModal" class="btn btn-secondary">取消</button>
              <button type="submit" class="btn btn-primary">保存</button>
            </div>
          </form>
        </div>
      </div>
    </transition>

    <!-- 随机抽取模态框  -->
    <transition name="modal">
      <div v-if="showRandomModal" class="modal-overlay" @click.self="showRandomModal = false">
        <div class="modal-content">
          <h2>🎲 随机抽取设置</h2>
          <div v-if="searchTerm.trim() !== ''" class="info-box">
            将从当前 <strong>{{ filteredSongs.length }}</strong> 首搜索结果中抽取。
          </div>
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

    <!-- 抽取结果模态框 -->
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
          <!-- 无结果提示 -->
          <div v-if="randomResults.length === 0" class="empty-state-small">
            <p>在指定的条件下没有找到可抽取的歌曲哦！</p>
          </div>
          <!-- 功能2: 新增“再次抽取”按钮 -->
          <div class="form-actions">
            <button @click="showResultModal = false" class="btn btn-secondary">关闭</button>
            <button @click="performRandomPick" class="btn btn-success">
              再次抽取
            </button>
          </div>
        </div>
      </div>
    </transition>

    <!-- 删除确认模态框 -->
    <transition name="modal">
      <div v-if="showDeleteConfirmModal" class="modal-overlay" @click.self="closeDeleteConfirmModal">
        <div class="modal-content">
          <h2>确认删除</h2>
          <p style="color: #555555">您确定要删除歌曲 "{{ songToDeleteTitle }}" 吗？此操作无法撤销。</p>
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
import { ref, computed, onMounted, watch, onUnmounted } from 'vue'
// 功能1: 假设后端增加了 ToggleBlacklist 方法
import { GetAllSongs, AddSong, UpdateSong, DeleteSong as BackendDeleteSong, ToggleFavorite, ToggleBlacklist, RandomPick } from '../wailsjs/go/main/App'

const searchTerm = ref('');
const allSongs = ref([])
const visibleSongs = ref([])
const loadMoreTrigger = ref(null)
const loading = ref(true);
const PAGE_SIZE = 20;
let observer = null;
const sortType = ref('title_asc')
const showFavoritesOnly = ref(false)
const showAddModal = ref(false)
const showRandomModal = ref(false)
const showResultModal = ref(false)
const editingItem = ref(null)
const randomResults = ref([])
const showDeleteConfirmModal = ref(false)
const songToDeleteId = ref(null)
const songToDeleteTitle = ref('')
const macaronColors = [
  '#FFB3BA', '#FFDFBA', '#FFFFBA', '#BAFFC9', '#BAE1FF',
  '#E0BBE4', '#FFB7D5', '#C7CEEA', '#B4E7F5', '#FED9B7',
  '#A0E7E5', '#F7C6C7', '#C9E4DE', '#FFD1DC', '#E4C1F9',
];
// 功能1: formData 中增加 isBlacklisted 默认值
const formData = ref({
  title: '', artist: '', level: 1, color: macaronColors[0], isFavorite: false, isBlacklisted: false
})
const randomOptions = ref({
  count: 1, minLevel: 0, maxLevel: 0, onlyFavorites: false
})

const filteredSongs = computed(() => {
  let result = [...allSongs.value]
  if (searchTerm.value.trim() !== '') {
    const lowerCaseSearch = searchTerm.value.toLowerCase();
    result = result.filter(s =>
        s.title.toLowerCase().includes(lowerCaseSearch) ||
        (s.artist && s.artist.toLowerCase().includes(lowerCaseSearch))
    );
  }
  if (showFavoritesOnly.value) {
    result = result.filter(s => s.isFavorite)
  }
  switch (sortType.value) {
    case 'title_asc': result.sort((a, b) => a.title.localeCompare(b.title)); break;
    case 'title_desc': result.sort((a, b) => b.title.localeCompare(a.title)); break;
    case 'level_desc': result.sort((a, b) => (b.level || 0) - (a.level || 0)); break;
    case 'level_asc': result.sort((a, b) => (a.level || 0) - (b.level || 0)); break;
  }
  return result
})
const hasMoreSongs = computed(() => {
  return visibleSongs.value.length < filteredSongs.value.length
})


watch([filteredSongs], () => {
  resetAndLoadVisibleSongs()
})

async function performRandomPick() {
  try {
    // 如果有搜索词，则在前端从搜索结果中抽取
    if (searchTerm.value.trim() !== '') {
      // 1. 从当前搜索结果中筛选符合条件的歌曲
      // 功能1: 核心改动 - 首先过滤掉黑名单中的歌曲
      let potentialPicks = [...filteredSongs.value].filter(s => !s.isBlacklisted);

      if (randomOptions.value.onlyFavorites) {
        potentialPicks = potentialPicks.filter(s => s.isFavorite);
      }
      if (randomOptions.value.minLevel > 0) {
        potentialPicks = potentialPicks.filter(s => (s.level || 0) >= randomOptions.value.minLevel);
      }
      if (randomOptions.value.maxLevel > 0) {
        potentialPicks = potentialPicks.filter(s => (s.level || 0) <= randomOptions.value.maxLevel);
      }

      // 2. 打乱数组（Fisher-Yates shuffle）
      for (let i = potentialPicks.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [potentialPicks[i], potentialPicks[j]] = [potentialPicks[j], potentialPicks[i]];
      }

      // 3. 取出所需数量的结果
      randomResults.value = potentialPicks.slice(0, randomOptions.value.count);

    } else {
      // 如果没有搜索词，则调用后端进行全局抽取（原有逻辑）
      // 假设后端的 RandomPick 方法已经内置了排除黑名单歌曲的逻辑
      randomResults.value = await RandomPick(randomOptions.value);
    }

    // 功能2: 如果不是从结果页再次抽取，则需要打开模态框
    if (!showResultModal.value) {
      showRandomModal.value = false;
      showResultModal.value = true;
    }

  } catch (err) {
    console.error('抽取失败:', err);
  }
}


function loadNextPage() {
  if (!hasMoreSongs.value) return;
  const currentPage = Math.ceil(visibleSongs.value.length / PAGE_SIZE);
  const start = currentPage * PAGE_SIZE;
  const end = start + PAGE_SIZE;
  const newSongs = filteredSongs.value.slice(start, end);
  visibleSongs.value.push(...newSongs);
}

function resetAndLoadVisibleSongs() {
  visibleSongs.value = filteredSongs.value.slice(0, PAGE_SIZE);
}

function setupObserver() {
  observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting) {
          loadNextPage();
        }
      },
      { root: null, rootMargin: '0px', threshold: 0.1 }
  );
  if (loadMoreTrigger.value) {
    observer.observe(loadMoreTrigger.value);
  }
}

onMounted(async () => {
  await loadSongs()
  setupObserver()
})

onUnmounted(() => {
  if (observer) {
    observer.disconnect();
  }
})

async function loadSongs() {
  loading.value = true;
  try {
    allSongs.value = await GetAllSongs() || []
    resetAndLoadVisibleSongs()
  } catch (err) {
    console.error('加载歌曲失败:', err)
  } finally {
    loading.value = false;
  }
}

function openAddModal() { resetForm(); showAddModal.value = true; }
function editSong(song) {
  editingItem.value = { ...song };
  // 功能1: 编辑时载入 isBlacklisted 状态
  formData.value = {
    id: song.id, title: song.title, artist: song.artist,
    level: song.level || 1, color: song.color || macaronColors[0],
    isFavorite: song.isFavorite, isBlacklisted: song.isBlacklisted || false,
  };
  showAddModal.value = true;
}
async function saveSong() {
  const songToSave = { ...formData.value };
  try {
    if (editingItem.value) {
      await UpdateSong(songToSave);
    } else {
      await AddSong(songToSave);
    }
    await loadSongs();
    closeAddModal();
  } catch (err) { console.error('保存失败:', err); }
}
function deleteSong(song) {
  songToDeleteId.value = song.id;
  songToDeleteTitle.value = song.title;
  showDeleteConfirmModal.value = true;
}
async function confirmDelete() {
  if (songToDeleteId.value === null) return;
  try {
    await BackendDeleteSong(songToDeleteId.value);
    await loadSongs();
  } catch (err) {
    console.error('删除失败:', err);
  } finally {
    closeDeleteConfirmModal();
  }
}
function closeDeleteConfirmModal() {
  showDeleteConfirmModal.value = false;
  songToDeleteId.value = null;
  songToDeleteTitle.value = '';
}
async function toggleFavorite(id) {
  try {
    const song = allSongs.value.find(s => s.id === id);
    if (song) { song.isFavorite = !song.isFavorite; }
    await ToggleFavorite(id);
  } catch (err) { console.error('操作失败:', err); await loadSongs(); }
}

// 功能1: 新增切换黑名单状态的函数
async function toggleBlacklist(id) {
  try {
    // 乐观更新 UI，立即反馈
    const song = allSongs.value.find(s => s.id === id);
    if (song) {
      song.isBlacklisted = !song.isBlacklisted;
    }
    // 调用后端 API
    await ToggleBlacklist(id);
  } catch (err) {
    console.error('黑名单操作失败:', err);
    // 如果失败，重新加载数据以恢复到正确状态
    await loadSongs();
  }
}

function closeAddModal() {
  showAddModal.value = false;
  editingItem.value = null;
  resetForm();
}

function resetForm() {
  // 功能1: 重置表单时包含 isBlacklisted
  formData.value = {
    title: '', artist: '', level: 1,
    color: macaronColors[0], isFavorite: false, isBlacklisted: false
  };
}

function openRandomModal() {
  randomOptions.value.onlyFavorites = showFavoritesOnly.value;
  showRandomModal.value = true;
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
  color: #f0f0f0;
  background-color: #090615;
  background-image: linear-gradient(315deg, #080717 0%, #1f1a3d 74%, #161626 100%);
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 30px;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(12px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.title {
  font-size: 28px;
  color: white;
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 0;
  text-shadow: 0 0 10px rgba(255, 255, 255, 0.3);
}

.search-group {
  position: relative;
  flex-grow: 1;
  max-width: 400px;
}
.search-icon {
  position: absolute;
  left: 15px;
  top: 50%;
  transform: translateY(-50%);
  color: rgba(255, 255, 255, 0.6);
  pointer-events: none;
}
.search-input {
  width: 100%;
  padding: 8px 12px 8px 40px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 8px;
  background: rgba(255, 255, 255, 0.1);
  color: white;
  font-size: 14px;
  transition: all 0.3s ease;
}
.search-input::placeholder {
  color: rgba(255, 255, 255, 0.5);
}
.search-input:focus {
  outline: none;
  border-color: rgba(255, 255, 255, 0.5);
  background: rgba(255, 255, 255, 0.15);
  box-shadow: 0 0 15px rgba(76, 175, 80, 0.2);
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
  padding: 15px 20px;
  border: none;
  border-radius: 20px;
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
  background: linear-gradient(45deg, #4facfe 0%, #00f2fe 100%);
  color: white;
}

.btn-success {
  background: linear-gradient(45deg, #fa709a 0%, #fee140 100%);
  color: #333;
}

.btn-secondary {
  background: linear-gradient(45deg, #4facfe 0%, #00f2fe 100%);
  color: white;
}

.filter-bar {
  display: flex;
  gap: 20px;
  padding: 15px 30px;
  background: rgba(255, 255, 255, 0.03);
  backdrop-filter: blur(12px);
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
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 8px;
  background: rgba(255, 255, 255, 0.1);
  color: white;
  cursor: pointer;
}

.select option {
  background-color: #2c3e50;
  color: white;
}

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  padding: 8px;
  border-radius: 8px;
  transition: background-color 0.2s;
}
.checkbox-label:hover {
  background-color: rgba(255,255,255,0.1);
}

.main-content {
  flex: 1;
  overflow-y: auto;
  padding: 30px;
}

.songs-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 25px; /* 增加卡片间距 */
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
  transform: translateY(-8px) scale(1.03);
  box-shadow: 3px 8px 30px rgb(161, 167, 197);
}

.card-overlay {
  position: absolute;
  inset: 0;
  background: linear-gradient(to top, rgba(0, 0, 0, 0.8) 0%, rgba(0, 0, 0, 0.1) 60%, rgba(0, 0, 0, 0) 100%);
  padding: 20px;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
}

.favorite-btn {
  position: absolute;
  top: 15px;
  right: 15px;
  background: rgba(0, 0, 0, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 50%;
  width: 40px;
  height: 40px;
  font-size: 20px;
  color: white;
  cursor: pointer;
  transition: all 0.3s;
  backdrop-filter: blur(5px);
  display: flex;
  align-items: center;
  justify-content: center;
}

.favorite-btn:hover {
  transform: scale(1.2);
  background-color: rgba(0, 0, 0, 0.6);
}

.favorite-btn.active {
  background: rgba(255, 215, 0, 0.8);
  color: white;
  animation: pulse 0.5s;
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.2); }
}

.card-content {
  color: white;
}

.song-title {
  font-size: 22px;
  font-weight: bold;
  margin: 0 0 5px 0;
  text-shadow: 2px 2px 6px rgba(0, 0, 0, 0.8);
}

.song-artist {
  font-size: 14px;
  opacity: 0.8;
  margin: 0 0 15px 0;
  text-shadow: 1px 1px 4px rgba(0, 0, 0, 0.7);
}

.song-meta {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.level-badge {
  padding: 5px 12px;
  border-radius: 15px;
  font-size: 12px;
  font-weight: bold;
  backdrop-filter: blur(5px);
  background: rgba(0, 0, 0, 0.5);
  color: white;
  border: 1px solid rgba(255,255,255,0.2);
}

.card-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  opacity: 0; /* 默认隐藏 */
  transform: translateY(10px); /* 默认下移 */
  transition: opacity 0.3s, transform 0.3s;
}

.song-card:hover .card-actions {
  opacity: 1; /* 悬浮时显示 */
  transform: translateY(0);
}


.btn-icon {
  background: rgba(255, 255, 255, 0.15);
  border: 1px solid rgba(255,255,255,0.2);
  border-radius: 50%;
  width: 36px;
  height: 36px;
  font-size: 16px;
  cursor: pointer;
  transition: all 0.3s;
  backdrop-filter: blur(5px);
  display: flex;
  align-items: center;
  justify-content: center;
}

.btn-icon:hover {
  background: rgba(255, 255, 255, 0.3);
  transform: scale(1.1);
}

.empty-state {
  text-align: center;
  padding: 60px 20px;
  color: rgba(255, 255, 255, 0.7);
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

/* ... 省略未修改的模态框和其它样式 ... */
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
  border-color: #4facfe;
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
  border-color: #4facfe;
  box-shadow: 0 0 0 2px white, 0 0 0 4px #4facfe;
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

.result-modal .btn-full {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
}

.load-more-trigger {
  height: 50px;
  width: 100%;
}

.info-box {
  background-color: #fffbe6;
  border: 1px solid #ffe58f;
  border-radius: 4px;
  padding: 8px 12px;
  margin-bottom: 16px;
  font-size: 14px;
  color: #8a6d3b;
}
.empty-state-small {
  text-align: center;
  padding: 20px;
  color: #888;
}


/* 为黑名单按钮添加激活样式 */
.blacklist-btn.active {
  opacity: 1;
  background-color: rgba(255, 99, 132, 0.2);
  border-radius: 50%;
  transform: scale(1.1);
}

.blacklist-btn {
  opacity: 0.5;
  transition: all 0.2s ease;
}

/* 调整再次抽取按钮，使其更好看 */
.result-modal .form-actions {
  display: flex;
  gap: 10px;
  margin-top: 20px;
}

.result-modal .form-actions button {
  flex: 1;
}

.result-modal .form-actions .btn-success span {
  margin-right: 8px;
  display: inline-block;
}

</style>