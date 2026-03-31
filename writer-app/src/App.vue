<script setup lang="ts">
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'
import localforage from 'localforage'
import Editor from './components/Editor.vue'

const editorRef = ref<any>(null)

// --- 沉浸模式与排版 ---
const isSidebarOpen = ref(true) 
const isTopbarOpen = ref(true)
const showTypography = ref(false)
const typography = ref({ fontSize: 18, lineHeight: 1.8, letterSpacing: 1, paragraphSpacing: 16, paddingX: 20, paddingBottom: 400, emptyLine: false, textIndent: 2 })

// --- 核心数据 ---
const books = ref<any[]>([]) 
const activeBookId = ref('') 
const catalog = ref<any[]>([]) 
const activeChapterId = ref('') 
const chapterContent = ref('') 
const isSaving = ref(false) 
const collapsedVolumes = ref<string[]>([])

const currentChapter = computed(() => catalog.value.find(c => c.id === activeChapterId.value))
const wordCount = computed(() => chapterContent.value.replace(/<[^>]+>/g, '').length)

// --- 自定义弹窗引擎 ---
const dialog = ref({ show: false, type: 'prompt', title: '', value: '', resolve: null as any })
const customPrompt = (title: string, defaultVal = '') => new Promise<string | null>(resolve => { dialog.value = { show: true, type: 'prompt', title, value: defaultVal, resolve } })
const customConfirm = (title: string) => new Promise<boolean>(resolve => { dialog.value = { show: true, type: 'confirm', title, value: '', resolve } })
const customAlert = (title: string) => new Promise<void>(resolve => { dialog.value = { show: true, type: 'alert', title, value: '', resolve } })
const closeDialog = (isOk: boolean) => { dialog.value.show = false; if (dialog.value.type === 'prompt') dialog.value.resolve(isOk ? dialog.value.value.trim() : null); else dialog.value.resolve(isOk) }

// --- 本地备份 ---
const showLocalManager = ref(false)
const exportLocalData = async () => {
  const data: any = { books: books.value, catalog: catalog.value, contents: {} }
  const keys = await localforage.keys()
  for (const k of keys) { if (k.startsWith('content_')) data.contents[k] = await localforage.getItem(k) }
  const blob = new Blob([JSON.stringify(data)], { type: 'application/json' })
  const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download = `小说本地备份_${new Date().toLocaleDateString().replace(/\//g, '')}.json`; a.click(); customAlert('✅ 备份文件已下载！')
}
const importLocalData = (e: any) => {
  const file = e.target.files[0]; if (!file) return
  const reader = new FileReader()
  reader.onload = async (ev: any) => {
    try {
      const data = JSON.parse(ev.target.result)
      if (data.books && data.catalog) {
        books.value = data.books; catalog.value = data.catalog
        for (const k in data.contents) await localforage.setItem(k, data.contents[k])
        await localforage.setItem('books', books.value); await localforage.setItem('catalog', catalog.value)
        chapterContent.value = (await localforage.getItem(`content_${activeChapterId.value}`)) || ''
        showLocalManager.value = false; customAlert('✅ 恢复成功！')
      }
    } catch (err) { customAlert('❌ 文件格式错误') }
  }
  reader.readAsText(file)
}

// --- 时光机快照 ---
const showHistory = ref(false)
const currentHistoryList = ref<any[]>([])
const activeHistoryItem = ref<any>(null)
let historyTimer: any = null
const startHistoryEngine = () => {
  historyTimer = setInterval(async () => {
    if (!activeChapterId.value || !chapterContent.value) return
    const histKey = `history_${activeChapterId.value}`
    const hist: any[] = (await localforage.getItem(histKey)) || []
    if (hist.length === 0 || hist[0].content !== chapterContent.value) {
      hist.unshift({ time: Date.now(), content: chapterContent.value })
      if (hist.length > 50) hist.length = 50 
      await localforage.setItem(histKey, hist)
    }
  }, 60000) 
}
const openHistoryModal = async () => { if (!activeChapterId.value) return; currentHistoryList.value = (await localforage.getItem(`history_${activeChapterId.value}`)) || []; activeHistoryItem.value = currentHistoryList.value[0] || null; showHistory.value = true }
const restoreHistory = async () => { if (!activeHistoryItem.value) return; const ok = await customConfirm('确认恢复到该版本？\n(未保存内容可按撤回找回)'); if (ok) { editorRef.value?.syncContentWithHistory(activeHistoryItem.value.content); showHistory.value = false } }

// --- Gitee 白嫖云同步引擎 ---
const showSettings = ref(false)
const isSyncing = ref(false) 
const syncMessage = ref('')  
const cloudConfig = ref({ token: '', owner: '', repo: 'writer-sync' })

const utoa = (str: string) => btoa(unescape(encodeURIComponent(str)))
const atou = (str: string) => decodeURIComponent(escape(atob(str)))

const getGiteeFile = async () => {
  const { token, owner, repo } = cloudConfig.value
  const url = `https://gitee.com/api/v5/repos/${owner}/${repo}/contents/sync_data.json?access_token=${token}`
  const res = await fetch(url)
  if (!res.ok) {
    if (res.status === 404) return null
    throw new Error(`网络错误或配置异常 (HTTP ${res.status})`)
  }
  return await res.json()
}

// 🔥 新增：云端连接诊断雷达 🔥
const testCloudConnection = async () => {
  const { token, owner, repo } = cloudConfig.value
  if (!token || !owner || !repo) return customAlert('请填写完整的 3 项配置后再检测！')
  
  try {
    const url = `https://gitee.com/api/v5/repos/${owner}/${repo}?access_token=${token}`
    const res = await fetch(url)
    if (res.ok) {
      customAlert('✅ 连接极其通畅！你的仓库状态完美，可以随时进行推送和拉取。')
    } else if (res.status === 401 || res.status === 403) {
      customAlert('❌ 凭证错误：私人令牌 (Token) 错误或已过期，请去 Gitee 重新生成！')
    } else if (res.status === 404) {
      customAlert('❌ 找不到仓库：请检查你的“空间地址”和“仓库名”是否拼错，或者你是否在 Gitee 上成功创建了这个私有仓库！')
    } else {
      customAlert(`❌ 未知错误：HTTP ${res.status}`)
    }
  } catch (err: any) {
    customAlert(`❌ 网络死锁：无法连接到 Gitee 接口，请检查本地网络环境！`)
  }
}

const forcePushToCloud = async () => { 
  if (!cloudConfig.value.token || !cloudConfig.value.owner) return customAlert('请先在左下角配置 Gitee 凭证！')
  isSyncing.value = true; syncMessage.value = '打包推送中...'
  try {
    const dataObj: any = { books: books.value, catalog: catalog.value, contents: {} }
    const keys = await localforage.keys()
    for (const key of keys) { if (key.startsWith('content_')) dataObj.contents[key] = await localforage.getItem(key) }
    
    const contentStr = JSON.stringify(dataObj)
    const base64Content = utoa(contentStr)

    let sha = ''
    try {
      const fileInfo = await getGiteeFile()
      if (fileInfo && fileInfo.sha) sha = fileInfo.sha
    } catch (e: any) { throw new Error(`获取旧数据失败：${e.message}`) }

    const { token, owner, repo } = cloudConfig.value
    const url = `https://gitee.com/api/v5/repos/${owner}/${repo}/contents/sync_data.json`
    const body: any = { access_token: token, content: base64Content, message: `Sync from App ${new Date().toLocaleString()}` }
    if (sha) body.sha = sha 
    
    const pushRes = await fetch(url, { method: sha ? 'PUT' : 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(body) })
    if (!pushRes.ok) throw new Error(`推送被拒绝 (HTTP ${pushRes.status})`)

    syncMessage.value = '☁️ 成功！'; setTimeout(() => syncMessage.value = '', 3000)
    customAlert('✅ 数据已成功极速推送到云端！') // 🔥 强制反馈
  } catch (err: any) { 
    syncMessage.value = '失败'; setTimeout(() => syncMessage.value = '', 3000)
    customAlert(`❌ 推送失败：${err.message}`) // 🔥 强制抛错
  }
  finally { isSyncing.value = false }
}

const forceSyncFromCloud = async () => { 
  if (!cloudConfig.value.token || !cloudConfig.value.owner) return customAlert('请先在左下角配置 Gitee 凭证！')
  const ok = await customConfirm('警告：拉取将完全覆盖本地进度！确认吗？')
  if (!ok) return
  isSyncing.value = true; syncMessage.value = '极速拉取中...'
  try {
    const fileInfo = await getGiteeFile()
    if (!fileInfo || !fileInfo.content) { 
      syncMessage.value = ''; customAlert('⚠️ 云端是一个空仓库，暂无数据可拉取！')
      return 
    }

    const jsonStr = atou(fileInfo.content)
    const data = JSON.parse(jsonStr)

    if (data.books && data.catalog) {
      books.value = data.books; catalog.value = data.catalog
      await localforage.setItem('books', JSON.parse(JSON.stringify(books.value)))
      await localforage.setItem('catalog', JSON.parse(JSON.stringify(catalog.value)))
      
      for (const k in data.contents) {
        await localforage.setItem(k, data.contents[k])
        if (k === `content_${activeChapterId.value}`) editorRef.value?.syncContentWithHistory(data.contents[k])
      }
    }
    syncMessage.value = '✅ 成功！'; setTimeout(() => syncMessage.value = '', 3000)
    customAlert('✅ 完美同步！本地数据已与云端对齐。') // 🔥 强制反馈
  } catch (err: any) { 
    syncMessage.value = '拉取失败'; setTimeout(() => syncMessage.value = '', 3000)
    customAlert(`❌ 拉取失败：${err.message}`) // 🔥 强制抛错
  }
  finally { isSyncing.value = false }
}

const saveCloudConfig = async () => {
  await localforage.setItem('cloudConfig', JSON.parse(JSON.stringify(cloudConfig.value)))
  customAlert('✅ Gitee 云端配置已保存！你可以点击左侧按钮【检测连接】测试一下。')
}

// --- 悬浮按钮自动隐藏逻辑 ---
const showToggles = ref(true)
let toggleTimer: any = null

const resetToggleTimer = () => {
  showToggles.value = true
  if (toggleTimer) clearTimeout(toggleTimer)
  toggleTimer = setTimeout(() => {
    showToggles.value = false
  }, 2000) // 2秒无操作自动隐藏
}

// --- 初始化与加载 ---
onMounted(async () => {
  resetToggleTimer() // 初始化启动隐藏倒计时
  const [b, c, cc, ty] = await Promise.all([localforage.getItem('books'), localforage.getItem('catalog'), localforage.getItem('cloudConfig'), localforage.getItem('typography')])
  if (cc) cloudConfig.value = { ...cloudConfig.value, ...(cc as any) }
  if (ty) typography.value = { ...typography.value, ...(ty as any) }

  if (b && (b as any).length > 0) {
    books.value = b as any; catalog.value = (c as any) || []; activeBookId.value = books.value[0].id
  } else {
    const bId = crypto.randomUUID(), vId = crypto.randomUUID(), c1Id = crypto.randomUUID()
    books.value.push({ id: bId, title: '【必读】国内极速免费云教程' })
    catalog.value.push({ id: vId, bookId: bId, type: 'volume', title: '云端与防丢' }, { id: c1Id, bookId: bId, type: 'chapter', title: '如何搭建 Gitee 同步' })
    
    await localforage.setItem(`content_${c1Id}`, `<p>由于国内外云数据库纷纷收费，我们利用 <b>Gitee（码云）</b> 打造免费无限云盘！操作极简：</p><p>1. 去 <a href="https://gitee.com/" target="_blank">Gitee 官网</a> 登录。</p><p>2. 点击右上角新建仓库。仓库名称填 <b>writer-sync</b>，权限设为 <b>【私有】</b>，勾选“初始化 readme”，然后创建。</p><p>3. 点击头像 -> 设置 -> 私人令牌 -> 生成新令牌。勾选 <b>projects</b> 权限，提交，得到一串很长的 <b>Token</b>。</p><p>4. 点击本软件左下角的 <b>云配置</b>，填入你的空间地址名（通常是拼音）、仓库名（writer-sync），粘贴 Token。</p><p>5. 点击<b>【检测连接】</b>，通了以后，就可以随时用右上角的云朵按钮推送拉取了！</p>`)
    
    activeBookId.value = bId; activeChapterId.value = c1Id
  }
  if (window.innerWidth < 768) isSidebarOpen.value = false
  startHistoryEngine() 
})
onBeforeUnmount(() => clearInterval(historyTimer))

watch(books, (val) => localforage.setItem('books', JSON.parse(JSON.stringify(val))), { deep: true })
watch(catalog, (val) => localforage.setItem('catalog', JSON.parse(JSON.stringify(val))), { deep: true })
watch(typography, (val) => localforage.setItem('typography', JSON.parse(JSON.stringify(val))), { deep: true })
watch(activeChapterId, async (newId) => { if (!newId) return; chapterContent.value = ((await localforage.getItem(`content_${newId}`)) as string) || '<p></p>'; if (window.innerWidth < 768) isSidebarOpen.value = false })

let saveTimeout: any = null
const onContentUpdate = (newHTML: string) => {
  chapterContent.value = newHTML; isSaving.value = true; clearTimeout(saveTimeout)
  saveTimeout = setTimeout(async () => { if (activeChapterId.value) { await localforage.setItem(`content_${activeChapterId.value}`, newHTML); isSaving.value = false } }, 2000)
}

const autoFormatText = () => { if (!activeChapterId.value) return; let html = chapterContent.value; html = html.replace(/<p><\/p>/g, '').replace(/<p><br><\/p>/g, '').replace(/<p>(&nbsp;|\s)+/g, '<p>').replace(/　/g, ''); editorRef.value?.syncContentWithHistory(html) }
const createBook = async () => { const t = await customPrompt('请输入新书名称：'); if (t) { const id = crypto.randomUUID(); books.value.push({ id, title: t }); activeBookId.value = id } }
const createVolume = async () => { if (!activeBookId.value) return; const t = await customPrompt('请输入分卷名称：'); if (t) catalog.value.push({ id: crypto.randomUUID(), bookId: activeBookId.value, type: 'volume', title: t }) }
const createChapter = async () => { if (!activeBookId.value) return; const t = await customPrompt('请输入章节名称：'); if (t) { const id = crypto.randomUUID(); catalog.value.push({ id, bookId: activeBookId.value, type: 'chapter', title: t }); activeChapterId.value = id } }
const renameBook = async () => { const b = books.value.find(b => b.id === activeBookId.value); if (b) { const t = await customPrompt('重命名书籍为：', b.title); if (t) b.title = t } }
const renameItem = async (i: any) => { const t = await customPrompt('重命名为：', i.title); if (t) i.title = t }
const toggleVolume = (volId: string) => { if (collapsedVolumes.value.includes(volId)) collapsedVolumes.value = collapsedVolumes.value.filter(id => id !== volId); else collapsedVolumes.value.push(volId) }
const currentCatalog = computed(() => catalog.value.filter(item => item.bookId === activeBookId.value))
const visibleCatalog = computed(() => { const result = []; let currentVolId = ''; for (const item of currentCatalog.value) { if (item.type === 'volume') { currentVolId = item.id; result.push(item) } else { if (!collapsedVolumes.value.includes(currentVolId)) result.push(item) } } return result })
const saveSortedCatalog = (sc: any[]) => { catalog.value = [...catalog.value.filter(i => i.bookId !== activeBookId.value), ...sc] }
const moveChapter = (id: string, d: 'up' | 'down') => { let l = [...currentCatalog.value]; const i = l.findIndex(x => x.id === id); if (i === -1) return; if (d === 'up' && i > 0) [l[i - 1], l[i]] = [l[i], l[i - 1]]; else if (d === 'down' && i < l.length - 1) [l[i], l[i + 1]] = [l[i + 1], l[i]]; saveSortedCatalog(l) }
const moveVolume = (id: string, d: 'up' | 'down') => {
  let l = [...currentCatalog.value]; const v = l.findIndex(x => x.id === id); if (v === -1) return; let nv = l.length; for (let i = v + 1; i < l.length; i++) if (l[i].type === 'volume') { nv = i; break }
  const cb = l.slice(v, nv)
  if (d === 'up') { if (v === 0) return; let pv = 0; for (let i = v - 1; i >= 0; i--) if (l[i].type === 'volume') { pv = i; break }; saveSortedCatalog([...l.slice(0, pv), ...cb, ...l.slice(pv, v), ...l.slice(nv)]) }
  else if (d === 'down') { if (nv === l.length) return; let nnv = l.length; for (let i = nv + 1; i < l.length; i++) if (l[i].type === 'volume') { nnv = i; break }; saveSortedCatalog([...l.slice(0, v), ...l.slice(nv, nnv), ...cb, ...l.slice(nnv)]) }
}
</script>

<template>
  <div @mousemove="resetToggleTimer" @touchstart="resetToggleTimer" @click="resetToggleTimer" 
      style="padding-top: env(safe-area-inset-top, 24px);padding-bottom: env(safe-area-inset-bottom, 0px);" class="flex h-dvh w-screen bg-gray-50 text-gray-800 font-sans select-none relative overflow-hidden box-border">
    
    <!-- 全局自定义弹窗 -->
    <div v-if="dialog.show" class="fixed inset-0 bg-black/50 z-100 flex items-center justify-center p-4 backdrop-blur-sm">
      <div class="bg-white rounded-xl shadow-2xl w-full max-w-sm p-6 transform transition-all border border-gray-100">
        <h3 class="text-lg font-bold text-gray-800 mb-4">{{ dialog.title }}</h3>
        <input v-if="dialog.type === 'prompt'" v-model="dialog.value" type="text" class="w-full border-2 border-gray-200 focus:border-blue-500 rounded-lg p-3 text-base outline-none mb-6" @keyup.enter="closeDialog(true)" autofocus>
        <div class="flex justify-end gap-3 mt-4">
          <button v-if="dialog.type !== 'alert'" @click="closeDialog(false)" class="px-5 py-2 bg-gray-100 text-gray-600 rounded-lg font-bold hover:bg-gray-200">取消</button>
          <button @click="closeDialog(true)" class="px-5 py-2 bg-blue-600 text-white rounded-lg font-bold shadow-md hover:bg-blue-700">确认</button>
        </div>
      </div>
    </div>

    <!-- 替换为你原来的固定在 left-0 top-1/2 的按钮 -->
      <button @click="isSidebarOpen = !isSidebarOpen" 
        :class="showToggles ? 'opacity-100' : 'opacity-0 pointer-events-none'"
        class="transition-opacity duration-500 fixed left-0 top-1/2 -translate-y-1/2 w-6 h-12 bg-gray-400/30 hover:bg-gray-400/50 rounded-r-xl flex items-center justify-center text-gray-800 z-50 backdrop-blur-md shadow-lg">
        {{ isSidebarOpen ? '◀' : '▶' }}
      </button>
    <!-- 左侧目录 -->
    <div v-show="isSidebarOpen" class="absolute md:relative z-40 h-full w-72 bg-gray-100 border-r border-gray-200 flex flex-col shadow-2xl md:shadow-none transition-transform">
      <div class="p-4 border-b border-gray-200 flex flex-col gap-2 relative bg-gray-200/50">
        
        <div class="flex justify-between items-center pr-16">
          <p class="text-xs text-gray-500">当前书籍</p>
          <button @click="createBook" class="text-xs text-gray-600 font-bold hover:underline">＋新建书</button>
        </div>
        <div class="flex items-center gap-1">
          <!-- select 增加 min-w-0 -->
          <select v-model="activeBookId" class="flex-1 min-w-0 bg-white border border-gray-300 rounded p-1 text-sm font-bold outline-none cursor-pointer">
            <option v-for="book in books" :key="book.id" :value="book.id">{{ book.title }}</option>
          </select>
          <!-- button 增加 shrink-0 -->
          <button @click="renameBook" class="shrink-0 text-gray-400 p-1 bg-white rounded border border-gray-300 shadow-sm">✎</button>
        </div>
      </div>
      <div class="flex-1 overflow-y-auto p-2 pb-20">
        <div v-for="element in visibleCatalog" :key="element.id" class="mb-1">
          <div v-if="element.type === 'volume'" class="group px-2 py-1 mt-3 text-xs font-bold text-gray-500 flex justify-between items-center rounded hover:bg-gray-200">
            <span class="truncate pr-2 flex-1"><span v-if="collapsedVolumes.includes(element.id)" class="text-blue-500 mr-1">▶</span>{{ element.title }}</span>
            <div class="flex items-center gap-1">
              <button @click="renameItem(element)" class="text-gray-400 hover:text-black px-1">✎</button>
              <button @click="moveVolume(element.id, 'up')" class="text-gray-400 hover:text-blue-600 px-1 font-bold">↑</button>
              <button @click="moveVolume(element.id, 'down')" class="text-gray-400 hover:text-blue-600 px-1 font-bold">↓</button>
              <button @click="toggleVolume(element.id)" class="text-gray-400 hover:text-blue-600 px-1 font-bold">{{ collapsedVolumes.includes(element.id) ? '＋' : '－' }}</button>
            </div>
          </div>
          <div v-else class="group px-4 py-2 text-sm rounded cursor-pointer flex justify-between items-center" :class="activeChapterId === element.id ? 'bg-blue-100 text-blue-600 font-bold' : 'hover:bg-gray-200'">
            <span class="truncate pr-2 flex-1" @click="activeChapterId = element.id">{{ element.title }}</span>
            <div class="flex items-center gap-1">
              <button @click.stop="renameItem(element)" class="text-gray-400 hover:text-black px-1">✎</button>
              <button @click.stop="moveChapter(element.id, 'up')" class="text-gray-400 hover:text-blue-600 px-1 font-bold">↑</button>
              <button @click.stop="moveChapter(element.id, 'down')" class="text-gray-400 hover:text-blue-600 px-1 font-bold">↓</button>
            </div>
          </div>
        </div>
      </div>
      <div class="p-4 border-t border-gray-200 flex justify-between items-center bg-gray-100 absolute bottom-0 w-full">
        <div class="flex gap-2">
          <button @click="showLocalManager = true" class="text-sm text-gray-400 hover:text-gray-800" title="本地数据备份">📁</button>
          <button @click="showSettings = true" class="text-sm text-gray-400 hover:text-gray-800">  ☁云配置</button>
        </div>
        <div class="flex gap-3">
          <button @click="createVolume" class="text-sm text-gray-600 font-bold">＋卷</button>
          <button @click="createChapter" class="text-sm text-gray-600 font-bold">＋章</button>
        </div>
      </div>
    </div>

    <!-- 右侧：编辑区 -->
    <div class="flex-1 flex flex-col bg-[#fdfcf8] z-0 relative h-full">
      <!-- 替换为你原来的固定在 top-0 left-1/2 的按钮 -->
        <button @click="isTopbarOpen = !isTopbarOpen" 
          :class="showToggles ? 'opacity-100' : 'opacity-0 pointer-events-none'"
          class="transition-opacity duration-500 absolute top-0 left-1/2 -translate-x-1/2 w-16 h-5 bg-gray-400/30 hover:bg-gray-400/50 rounded-b-xl flex items-center justify-center text-gray-800 z-50 backdrop-blur-md shadow-lg text-xs">
          {{ isTopbarOpen ? '▲' : '▼' }}
        </button>

      <div v-show="isTopbarOpen" class="border-b border-gray-200 flex flex-wrap items-center p-2 px-4 gap-2 bg-white relative z-20 shadow-sm shrink-0">
        
        <h2 class="text-lg font-bold text-gray-800 truncate max-w-36 md:max-w-xs">{{ currentChapter ? currentChapter.title : '未选择' }}</h2>
        
        <div class="ml-auto flex items-center gap-2 md:gap-4 flex-wrap justify-end">
          <button @click="openHistoryModal" class="text-sm bg-gray-100 text-gray-700 px-2 py-1 rounded font-bold hover:bg-purple-100">历史版本</button>
          <button @click="autoFormatText" class="text-sm bg-gray-100 text-gray-700 px-2 py-1 rounded font-bold hover:bg-blue-100">智能排版</button>
          <button @click="showTypography = true" class="text-sm bg-gray-100 text-gray-700 px-2 py-1 rounded font-bold hover:bg-gray-200">排版设置</button>
          <span class="hidden md:inline text-gray-300">|</span>
          <button @click="editorRef?.undo()" :disabled="!editorRef?.canUndo" class="text-gray-500 disabled:opacity-20 px-1 font-bold text-lg">↶</button>
          <button @click="editorRef?.redo()" :disabled="!editorRef?.canRedo" class="text-gray-500 disabled:opacity-20 px-1 font-bold text-lg">↷</button>
          <span class="hidden md:inline text-gray-300">|</span>

          <div class="flex rounded border border-gray-300 overflow-hidden">
            <button @click="forcePushToCloud" class="text-xs md:text-sm bg-white text-gray-700 px-2 py-1 font-bold border-r border-gray-300">☁️ 推送</button>
            <button @click="forceSyncFromCloud" class="text-xs md:text-sm bg-gray-100 text-gray-700 px-2 py-1 font-bold">🔃 拉取</button>
          </div>
          <span class="text-xs text-gray-400 w-full md:w-auto text-right">{{ wordCount }}字 | {{ isSaving ? '保存中...' : '已保存' }}</span>
        </div>
      </div>

      <div id="editor-scroll-box" class="flex-1 overflow-y-auto flex justify-center relative scroll-smooth">
        <div class="w-full max-w-3xl mt-4 md:mt-10" v-if="activeChapterId">
          <Editor ref="editorRef" :modelValue="chapterContent" :typography="typography" @update:modelValue="onContentUpdate" />
        </div>
      </div>
    </div>

    <!-- 本地文件夹管理 -->
    <div v-if="showLocalManager" class="fixed inset-0 bg-black/40 flex items-center justify-center z-50 p-4">
      <div class="bg-white rounded-lg shadow-xl w-96 p-6">
        <h3 class="text-lg font-bold text-gray-800 mb-2">📁 本地数据备份</h3>
        <p class="text-xs text-gray-500 mb-6 leading-relaxed">如果要更换设备或手动保管，可将全部数据打包下载为 JSON 文件。</p>
        <div class="space-y-4 border-t pt-4">
          <button @click="exportLocalData" class="w-full py-3 bg-green-50 text-green-700 border border-green-200 rounded font-bold hover:bg-green-100">⬇️ 下载全局备份文件</button>
          <div class="relative w-full">
            <button class="w-full py-3 bg-red-50 text-red-700 border border-red-200 rounded font-bold hover:bg-red-100">⬆️ 从备份恢复数据 (覆盖)</button>
            <input type="file" accept=".json" class="absolute inset-0 opacity-0 cursor-pointer" @change="importLocalData">
          </div>
        </div>
        <div class="flex justify-end mt-6"><button @click="showLocalManager = false" class="px-4 py-2 bg-gray-100 text-gray-700 rounded font-bold">关闭</button></div>
      </div>
    </div>

    <!-- 历史版本弹窗 -->
    <div v-if="showHistory" class="fixed inset-0 bg-black/60 flex items-center justify-center z-50 p-2 md:p-6">
      <div class="bg-white rounded-lg shadow-2xl w-full max-w-5xl h-[85vh] flex flex-col overflow-hidden">
        <div class="p-4 border-b flex justify-between items-center bg-gray-50 shrink-0">
          <h3 class="text-lg font-bold text-gray-800">📜 历史快照</h3>
          <button @click="showHistory = false" class="text-gray-500 hover:text-black font-bold text-xl px-2">×</button>
        </div>
        <div class="flex-1 flex flex-col md:flex-row overflow-hidden bg-white">
          <div class="w-full md:w-1/3 border-b md:border-b-0 md:border-r border-gray-200 overflow-y-auto bg-gray-50 p-2 shrink-0 max-h-48 md:max-h-full">
            <div v-if="currentHistoryList.length === 0" class="text-center text-gray-400 mt-6 text-sm">暂无历史快照</div>
            <div v-for="(item, idx) in currentHistoryList" :key="item.time" @click="activeHistoryItem = item" class="p-3 border cursor-pointer rounded mb-2 transition-colors" :class="activeHistoryItem?.time === item.time ? 'bg-blue-50 border-blue-300 shadow-sm' : 'bg-white border-gray-200 hover:bg-gray-100'">
              <div class="flex justify-between items-center"><span class="font-bold text-sm text-gray-800">快照 {{ currentHistoryList.length - idx }}</span><span class="text-[10px] text-gray-400">{{ new Date(item.time).toLocaleTimeString() }}</span></div>
              <div class="text-xs text-gray-400 mt-2 truncate">{{ item.content.replace(/<[^>]+>/g, '').substring(0, 18) || '空内容' }}...</div>
            </div>
          </div>
          <div class="w-full md:w-2/3 flex flex-col bg-[#fdfcf8] flex-1 min-h-0">
            <div class="p-2 md:p-3 border-b border-gray-200 bg-white flex justify-between items-center shrink-0">
              <span class="text-sm text-gray-500 font-bold">预览 <span v-if="activeHistoryItem" class="hidden md:inline">({{ new Date(activeHistoryItem.time).toLocaleString() }})</span></span>
              <button @click="restoreHistory" :disabled="!activeHistoryItem" class="px-2 py-1 text-xs md:px-4 md:py-1.5 md:text-sm bg-purple-600 text-white rounded font-bold shadow disabled:opacity-50 hover:bg-purple-700 whitespace-nowrap">↶ 恢复到此版本</button>
            </div>
            <div class="flex-1 overflow-y-auto p-6 md:p-10 history-preview text-gray-700" v-if="activeHistoryItem" v-html="activeHistoryItem.content"></div>
          </div>
        </div>
      </div>
    </div>

    <!-- 排版设置 -->
    <div v-if="showTypography" class="fixed inset-0 bg-black/40 flex items-center justify-center z-50 p-4">
      <div class="bg-white rounded-lg shadow-xl w-full max-w-md p-6 max-h-[90vh] overflow-y-auto">
        <h3 class="text-lg font-bold text-gray-800 mb-4 border-b pb-2">排版设置</h3>
        <div class="space-y-4">
          <div class="flex justify-between items-center"><label class="text-sm font-bold text-gray-700">首行缩进</label><input type="range" v-model="typography.textIndent" min="0" max="2" step="1" class="w-1/3"></div>
          <div class="flex justify-between items-center"><label class="text-sm font-bold text-gray-700">字体大小</label><input type="range" v-model="typography.fontSize" min="12" max="32" class="w-1/3"></div>
          <div class="flex justify-between items-center"><label class="text-sm font-bold text-gray-700">行间距</label><input type="range" v-model="typography.lineHeight" min="1.0" max="3.0" step="0.1" class="w-1/3"></div>
          <div class="flex justify-between items-center"><label class="text-sm font-bold text-gray-700">段间距</label><input type="range" v-model="typography.paragraphSpacing" min="0" max="50" class="w-1/3"></div>
          <div class="flex justify-between items-center"><label class="text-sm font-bold text-gray-700">字间距</label><input type="range" v-model="typography.letterSpacing" min="0" max="10" step="0.5" class="w-1/3"></div>
          <div class="flex justify-between items-center"><label class="text-sm font-bold text-gray-700">左右边距</label><input type="range" v-model="typography.paddingX" min="0" max="100" class="w-1/3"></div>
          <div class="flex justify-between items-center"><label class="text-sm font-bold text-gray-700">底部安全滚动</label><input type="range" v-model="typography.paddingBottom" min="100" max="800" step="50" class="w-1/3"></div>
        </div>
        <div class="flex justify-end mt-6"><button @click="showTypography = false" class="px-6 py-2 bg-blue-600 text-white rounded font-bold shadow">完成</button></div>
      </div>
    </div>

    <!-- 🔥 Gitee 云配置 (增加极其明显的检测按钮) 🔥 -->
    <div v-if="showSettings" class="fixed inset-0 bg-black/40 flex items-center justify-center z-50 p-4">
      <div class="bg-white rounded-lg shadow-xl w-96 p-6">
        <h3 class="text-lg font-bold text-gray-800 mb-4 border-b pb-2">Gitee 国内免费云配置</h3>
        <div class="space-y-4">
          <div><label class="block text-xs text-gray-500 mb-1">Gitee 空间地址 (比如账号拼音)</label><input v-model="cloudConfig.owner" type="text" placeholder="例如: zhangsan" class="w-full border p-2 text-sm rounded"></div>
          <div><label class="block text-xs text-gray-500 mb-1">Gitee 仓库名</label><input v-model="cloudConfig.repo" type="text" placeholder="建议保持为: writer-sync" class="w-full border p-2 text-sm rounded"></div>
          <div><label class="block text-xs text-gray-500 mb-1">私人令牌 (Token)</label><input v-model="cloudConfig.token" type="password" placeholder="获取自 Gitee 设置页" class="w-full border p-2 text-sm rounded"></div>
        </div>
        
        <div class="flex justify-end gap-3 mt-6">
          <!-- 🔥 新增的雷达检测按钮 🔥 -->
          <button @click="testCloudConnection" class="px-4 py-2 bg-yellow-50 text-yellow-700 rounded font-bold border border-yellow-200 hover:bg-yellow-100">检测连接</button>
          
          <button @click="showSettings = false" class="px-4 py-2 bg-gray-100 text-gray-700 rounded font-bold">关闭</button>
          <button @click="saveCloudConfig" class="px-4 py-2 bg-blue-600 text-white rounded font-bold shadow">保存</button>
        </div>
      </div>
    </div>

  </div>
</template>

<style>
.history-preview p { text-indent: 2em; margin-bottom: 1em; line-height: 1.8; font-size: 1.1rem; }
</style>