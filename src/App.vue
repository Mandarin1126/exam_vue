<script setup>
import { ref, reactive, computed ,onMounted} from 'vue'

onMounted(() => {
  // 1. 尝试从缓存恢复登录信息
  const savedUser = localStorage.getItem('user')
  if (savedUser) {
    try {
      const parsedUser = JSON.parse(savedUser)
      currentUser.value = parsedUser
      isLoggedIn.value = true
      
      // 新增：如果是管理员，刷新后立刻拉取用户列表
      if (parsedUser.role === 'ADMIN') {
        fetchUserList(); 
      }
      
    } catch (e) {
      localStorage.removeItem('user')
    }
  }

  // 2. 加载题目
  fetchQuestions()
})

// ================= 0. 配置区 =================
// 确保这里指向你的 SpringBoot 端口
const API_BASE = 'http://localhost:8080/api'

// 分类配置 (保持 Easy 风格)
const categories = [
  { id: 'politics', name: '政治理论', icon: '🚩', color: '#c0392b', bg: '#fadbd8' },
  { id: 'common',   name: '常识判断', icon: '🌍', color: '#e67e22', bg: '#fdebd0' },
  { id: 'verbal',   name: '言语理解', icon: '📝', color: '#2980b9', bg: '#d6eaf8' },
  { id: 'logic',    name: '判断推理', icon: '🧩', color: '#8e44ad', bg: '#ebdef0' },
  { id: 'math',     name: '数量关系', icon: '📐', color: '#27ae60', bg: '#d5f5e3' },
  { id: 'data',     name: '资料分析', icon: '📊', color: '#34495e', bg: '#d6dbdf' }
]

// ================= 1. 状态管理 =================
const isLoggedIn = ref(false)
const currentUser = ref({ id: null, username: '', role: '' })

// 🔐 登录/注册 表单数据
const isRegisterMode = ref(false) // false=登录模式, true=注册模式
const authForm = reactive({ username: '', password: '', confirmPassword: '' })

// 题目数据
const questions = ref([])
const currentQuestion = ref(null)
const selectedOption = ref(null)
const showResult = ref(false)
const commentList = ref([])
const myComment = ref('')
const showAdminPanel = ref(false)
const userList = ref([])
const auditModalVisible = ref(false)
const auditComments = ref([])
const auditTargetUser = ref('')

// 题目上传相关状态
const showUploadPanel = ref(false)
const uploadForm = reactive({
  categoryId: 'politics', // 改为使用分类ID
  title: '',
  options: ['A. ', 'B. ', 'C. ', 'D. '],
  answer: '',
  explanation: ''
})

// 管理员审核相关状态
const showReviewPanel = ref(false)
const pendingQuestions = ref([])
const myUploads = ref([])
const showMyUploadsPanel = ref(false)

// ================= 计算属性 =================
const sidebarGroups = computed(() => {
  return categories.map(cat => {
    const qs = questions.value.filter(q => q.type === cat.id)
    return { meta: cat, list: qs }
  }).filter(group => group.list.length > 0)
})

const currentCategoryInfo = computed(() => {
  if (!currentQuestion.value) return {}
  return categories.find(c => c.id === currentQuestion.value.type) || {}
})

// ================= 核心功能逻辑 =================

// 🚀 1. 登录功能
async function handleLogin() {
  if (!authForm.username || !authForm.password) return alert('请填写完整哦')
  try {
    const res = await fetch(`${API_BASE}/user/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username: authForm.username, password: authForm.password })
    })
    const data = await res.json()
    
    if (data.code === 200) {
      isLoggedIn.value = true
      currentUser.value = data.data // 获取后端返回的用户信息
      localStorage.setItem('user', JSON.stringify(data.data))
      
      if (currentUser.value.role === 'ADMIN') {
        fetchUserList()
        showAdminPanel.value = true
      } else {
        enterQuestion(questions.value[0])
      }
    } else {
      alert(`登录失败: ${data.msg}`)
    }
  } catch (e) { alert('服务器没反应，请检查后端是否启动') }
}

// ✨ 2. 注册功能 (对应你的后端代码)
async function handleRegister() {
  // 前端校验
  if (!authForm.username || !authForm.password) return alert('账号密码不能为空')
  if (authForm.password !== authForm.confirmPassword) return alert('两次输入的密码不一致！')

  try {
    const res = await fetch(`${API_BASE}/user/register`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ 
        username: authForm.username, 
        password: authForm.password,
      })
    })
    const data = await res.json()
    
    // 处理后端返回结果 (code 200 / 400 / 500)
    if (data.code === 200) {
      alert('🎉 注册成功！请登录体验。')
      // 注册成功后逻辑：切回登录模式，清空密码
      isRegisterMode.value = false 
      authForm.password = ''
      authForm.confirmPassword = ''
    } else {
      alert(`😢 注册失败: ${data.msg}`)
    }
  } catch (e) { alert('网络开小差了，注册请求发送失败') }
}

// 3. 退出
function handleLogout() {
  isLoggedIn.value = false
  currentUser.value = {}
  showAdminPanel.value = false
  currentQuestion.value = null
  authForm.username = ''
  authForm.password = ''
  commentList.value = []
  localStorage.removeItem('user')
}

// 4. 业务逻辑 (做题、评论等)
function enterQuestion(q) {
  currentQuestion.value = q
  selectedOption.value = null
  showResult.value = false
  commentList.value = []
  fetchComments(q.id)
}

async function fetchComments(qid) {
  try {
    const res = await fetch(`${API_BASE}/comment/list?questionId=${qid}`)
    const data = await res.json()
    if (data.code === 200) commentList.value = data.data
  } catch (e) {}
}

function submitAnswer() {
  if (!selectedOption.value) return alert('选一个答案呗')
  showResult.value = true
}

async function submitComment() {
  // 1. 校验
  if (!currentUser.value.id) return alert('请先登录');
  if (!myComment.value.trim()) return alert('内容不能为空');

  try {
    // 2. 发送请求
    const res = await fetch(`${API_BASE}/comment/add`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        userId: currentUser.value.id,      // 对应后端 Comment 实体中的 userId
        questionId: currentQuestion.value.id, // 对应后端 Comment 实体中的 questionId
        content: myComment.value           // 对应后端 Comment 实体中的 content
      })
    });
    
    const data = await res.json();
    
    // 3. 处理结果
    if (data.code === 200) {
      // 发送成功：清空输入框，重新拉取评论列表
      myComment.value = '';
      fetchComments(currentQuestion.value.id);
    } else {
      alert(data.msg); // 例如：内容不能为空
    }
  } catch (e) {
    console.error(e);
    alert('评论发送失败');
  }
}

async function handleAction(comment, type) {
  // 1. 没登录先拦截
  if (!currentUser.value.id) return alert('请先登录后再操作！');

  try {
    // 2. 发送请求给后端
    const res = await fetch(`${API_BASE}/comment/action`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        userId: currentUser.value.id,
        commentId: comment.id,
        type: type 
      })
    });
    
    const data = await res.json();

    // 3. 判断后端返回结果
    if (data.code === 200) {
      // ✅ 成功：更新前端显示的数字
      if (type === 1) {
        // 只有点赞才更新界面上的数字，防止界面乱跳
        comment.likeCount = (comment.likeCount || 0) + 1;
      } else if (type === 3) {
        alert('举报成功，管理员会尽快处理');
      }
      else if(type===2){
        comment.dislikeCount = (comment.dislikeCount || 0) + 1
      }
    } else {
      // ❌ 失败（例如 code=400 "您已操作过"）：弹窗提示用户
      alert(data.msg); 
    }
  } catch (e) {
    console.error(e);
    alert('网络请求失败');
  }
}

async function fetchUserList() {
  try {
    const res = await fetch(`${API_BASE}/user/list`)
    const data = await res.json()
    if (data.code === 200) userList.value = data.data
  } catch (e) {}
}

async function changeStatus(user, newStatus) {
  if (!confirm('确定修改状态?')) return
  try {
    await fetch(`${API_BASE}/user/status`, {
      method: 'POST', headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({ id: user.id, status: newStatus })
    })
    fetchUserList()
  } catch (e) {}
}

async function openAudit(user) {
  auditTargetUser.value = user.username
  try {
    const res = await fetch(`${API_BASE}/user/comments/${user.id}`)
    const data = await res.json()
    if (data.code === 200) { auditComments.value = data.data; auditModalVisible.value = true }
  } catch (e) {}
}

async function deleteComment(comment) {
  if (!confirm('确定要删除这条评论吗？此操作不可恢复。')) return;

  try {
    const res = await fetch(`${API_BASE}/comment/delete`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ 
        id: comment.id,            // 评论ID
        userId: currentUser.value.id // 操作人ID
      })
    })
    const data = await res.json()

    if (data.code === 200) {
      // ✅ 删除成功
      // 为了体验流畅，直接在前端列表里移除它，不用重新刷新整个列表
      commentList.value = commentList.value.filter(item => item.id !== comment.id);
    } else {
      alert(data.msg); // 例如 "无权删除"
    }
  } catch (e) {
    console.error(e);
    alert('请求失败，请检查网络');
  }
}
// 从后端拉取题库
// ================= 题目上传功能 =================
async function uploadQuestion() {
  // 前端校验
  if (!uploadForm.title.trim()) return alert('题目内容不能为空')
  if (!uploadForm.answer.trim()) return alert('答案不能为空')

  // 处理选项（固定为选择题）
  let processedOptions = uploadForm.options.filter(opt => opt.trim())
  if (processedOptions.length < 2) return alert('选择题至少需要2个选项')

  try {
    const res = await fetch(`${API_BASE}/question/upload`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        type: uploadForm.categoryId, // 使用分类ID作为type
        title: uploadForm.title,
        options: JSON.stringify(processedOptions),
        answer: uploadForm.answer,
        explanation: uploadForm.explanation,
        uploaderId: currentUser.value.id
      })
    })
    const data = await res.json()

    if (data.code === 200) {
      alert('🎉 题目上传成功，等待管理员审核')
      // 重置表单
      uploadForm.categoryId = 'politics'
      uploadForm.title = ''
      uploadForm.options = ['A. ', 'B. ', 'C. ', 'D. ']
      uploadForm.answer = ''
      uploadForm.explanation = ''
      showUploadPanel.value = false
      // 刷新我的上传记录
      if (showMyUploadsPanel.value) {
        fetchMyUploads()
      }
    } else {
      alert(`上传失败: ${data.msg}`)
    }
  } catch (e) {
    alert('网络开小差了，上传请求发送失败')
  }
}

// ================= 管理员审核功能 =================
async function fetchPendingQuestions() {
  try {
    const res = await fetch(`${API_BASE}/question/pending`)
    const data = await res.json()
    if (data.code === 200) pendingQuestions.value = data.data
  } catch (e) {
    console.error("获取待审核题目失败", e)
  }
}

async function reviewQuestion(questionId, reviewStatus) {
  const statusText = reviewStatus === 1 ? '通过' : '拒绝'
  if (!confirm(`确定要${statusText}这道题目吗？`)) return

  try {
    const res = await fetch(`${API_BASE}/question/review?id=${questionId}&reviewStatus=${reviewStatus}`, {
      method: 'POST'
    })
    const data = await res.json()

    if (data.code === 200) {
      alert(`题目${statusText}成功`)
      // 刷新待审核列表
      fetchPendingQuestions()
      // 刷新题目列表
      fetchQuestions()
    } else {
      alert(`操作失败: ${data.msg}`)
    }
  } catch (e) {
    alert('网络请求失败')
  }
}

// ================= 我的上传记录功能 =================
async function fetchMyUploads() {
  try {
    const res = await fetch(`${API_BASE}/question/my?uploaderId=${currentUser.value.id}`)
    const data = await res.json()
    if (data.code === 200) myUploads.value = data.data
  } catch (e) {
    console.error("获取我的上传记录失败", e)
  }
}

// ================= 辅助函数 =================
function addOption() {
  if (uploadForm.options.length < 8) {
    uploadForm.options.push(`${String.fromCharCode(65 + uploadForm.options.length)}. `)
  } else {
    alert('最多支持8个选项')
  }
}

function removeOption(index) {
  if (uploadForm.options.length > 2) {
    uploadForm.options.splice(index, 1)
    // 重新编号
    uploadForm.options.forEach((opt, i) => {
      uploadForm.options[i] = `${String.fromCharCode(65 + i)}. ${opt.substring(3)}`
    })
  } else {
    alert('至少需要2个选项')
  }
}

function getReviewStatusText(status) {
  switch(status) {
    case 0: return '待审核'
    case 1: return '已通过'
    case 2: return '已拒绝'
    default: return '未知'
  }
}

function getReviewStatusClass(status) {
  switch(status) {
    case 0: return 'status-pending'
    case 1: return 'status-approved'
    case 2: return 'status-rejected'
    default: return ''
  }
}

// 根据分类ID获取分类信息
function getCategoryById(categoryId) {
  return categories.find(c => c.id === categoryId) || categories[0]
}

// ================= 导航折叠功能 =================
// 存储每个分类的折叠状态
const collapsedCategories = ref({})

// 切换分类折叠状态
function toggleCategory(categoryId) {
  collapsedCategories.value[categoryId] = !collapsedCategories.value[categoryId]
}

// 检查分类是否折叠
function isCategoryCollapsed(categoryId) {
  return collapsedCategories.value[categoryId] || false
}

// 从后端拉取题库
async function fetchQuestions() {
  try {
    const res = await fetch(`${API_BASE}/question/list`)
    const data = await res.json()

    if (data.code === 200) {
      // 🔥 数据清洗流水线
      questions.value = data.data.map(q => {
        // 1. 处理选项：把字符串 "['A','B']" 转成数组 ['A','B']
        if (typeof q.options === 'string') {
          try {
            q.options = JSON.parse(q.options)
          } catch (e) {
            q.options = [] // 解析失败给个空数组
          }
        }

        // 2. 处理解析字段名：数据库叫 explanation，前端模板里用的 explain
        // 如果你不想改模板，就在这里赋值一下
        if (!q.explain && q.explanation) {
          q.explain = q.explanation
        }

        return q
      })

      // 默认选中第一题 (如果列表不为空)
      if (questions.value.length > 0) {
        enterQuestion(questions.value[0])
      }
    }
  } catch (e) {
    console.error("加载题库失败", e)
    alert("题库加载失败，请检查后端服务")
  }
}
</script>

<template>
  <div class="app-container">
    
    <header class="navbar">
      <div class="brand">
        <span class="logo-icon">⚡</span> EasyExam <span class="sub-brand"></span>
      </div>
      <div v-if="isLoggedIn" class="user-info">
        <div class="avatar-circle">{{ currentUser.username ? currentUser.username[0].toUpperCase() : 'U' }}</div>
        <div class="u-details">
          <span class="u-name">{{ currentUser.username }}</span>
          <span class="u-role">{{ currentUser.role === 'ADMIN' ? 'Admin' : 'User' }}</span>
        </div>
        <button v-if="currentUser.role === 'ADMIN'" @click="showAdminPanel = !showAdminPanel" class="nav-btn">{{ showAdminPanel ? '刷题' : '管理后台' }}</button>
        <button @click="showUploadPanel = true" class="nav-btn">📝 上传题目</button>
        <button @click="showMyUploadsPanel = true; fetchMyUploads()" class="nav-btn">📂 我的上传</button>
        <button @click="handleLogout" class="nav-btn ghost">退出</button>
      </div>
    </header>

    <div v-if="!isLoggedIn" class="login-wrapper">
      <div class="login-card">
        <div class="login-header">
          <h1>{{ isRegisterMode ? '🚀 创建新账号' : '👋 Hi, Welcome!' }}</h1>
          <p>{{ isRegisterMode ? '加入 EasyExam，开启挑战' : '登录你的账号继续刷题' }}</p>
        </div>
        
        <div class="input-group">
          <input v-model="authForm.username" placeholder="用户名 / 账号" />
          <input v-model="authForm.password" type="password" placeholder="密码" />
          
          <transition name="fade">
            <input v-if="isRegisterMode" v-model="authForm.confirmPassword" type="password" placeholder="再次确认密码" />
          </transition>
        </div>

        <button v-if="!isRegisterMode" @click="handleLogin" class="login-btn">
          登录 Login
        </button>
        <button v-else @click="handleRegister" class="login-btn register-color">
          立即注册 Sign Up
        </button>
        
        <div class="toggle-mode">
          <span v-if="!isRegisterMode">还没有账号？ <a @click="isRegisterMode = true">去注册一个</a></span>
          <span v-else>已有账号？ <a @click="isRegisterMode = false">返回登录</a></span>
        </div>
      </div>
    </div>

    <div v-else class="main-body">
      <div v-if="showAdminPanel && currentUser.role === 'ADMIN'" class="admin-panel">
        <!-- 管理面板标签切换 -->
        <div class="admin-tabs">
          <button :class="['tab-btn', { active: !showReviewPanel }]" @click="showReviewPanel = false">👥 用户管理</button>
          <button :class="['tab-btn', { active: showReviewPanel }]" @click="showReviewPanel = true; fetchPendingQuestions()">📋 题目审核</button>
        </div>

        <!-- 用户管理面板 -->
        <div v-if="!showReviewPanel">
          <div class="panel-header"><h3>🛡️ 用户管理</h3></div>
          <table class="data-table">
            <thead><tr><th>ID</th><th>用户</th><th>状态</th><th>操作</th></tr></thead>
            <tbody>
              <tr v-for="u in userList" :key="u.id">
                <td>#{{ u.id }}</td>
                <td>{{ u.username }} <span v-if="u.totalReportCount>0" class="tag-report">{{u.totalReportCount}}🔥</span></td>
                <td>{{ u.status }}</td>
                <td>
                  <button v-if="u.role!=='ADMIN'" @click="openAudit(u)" class="btn-mini">审计</button>
                  <button v-if="u.status!=='BANNED' && u.role!=='ADMIN'" @click="changeStatus(u, 'BANNED')" class="btn-mini danger">禁言</button>
                  <button v-if="u.status==='BANNED'" @click="changeStatus(u, 'NORMAL')" class="btn-mini success">解封</button>
                </td>
              </tr>
            </tbody>
          </table>
          <div v-if="auditModalVisible" class="modal-mask" @click.self="auditModalVisible=false">
            <div class="modal-box">
              <h4>用户 [{{ auditTargetUser }}] 发言</h4>

              <div v-for="c in auditComments" :key="c.id" class="audit-item">
                <div class="audit-content">{{ c.content }}</div>

                <div class="audit-info">
                  <span>🕒 {{ c.createTime ? c.createTime.replace('T', ' ') : '未知时间' }}</span>
                  <span style="margin-left: 15px;">📍 题目 #{{ c.questionId }}</span>
                </div>
              </div>
              <button @click="auditModalVisible=false" style="width:100%;margin-top:10px">关闭</button>
            </div>
          </div>
        </div>

        <!-- 题目审核面板 -->
        <div v-else>
          <div class="panel-header"><h3>📋 待审核题目列表</h3></div>
          <div v-if="pendingQuestions.length === 0" class="empty-state">暂无待审核题目</div>
          <div v-else class="review-list">
            <div v-for="q in pendingQuestions" :key="q.id" class="review-item">
              <div class="review-header">
                <span class="review-type">{{ getCategoryById(q.type).icon }} {{ getCategoryById(q.type).name }}</span>
                <span class="review-id">#{{ q.id }}</span>
              </div>
              <div class="review-title">{{ q.title }}</div>
              <div v-if="q.options && JSON.parse(q.options).length > 0" class="review-options">
                <div v-for="(opt, idx) in JSON.parse(q.options)" :key="idx" class="review-option">{{ opt }}</div>
              </div>
              <div class="review-meta">
                <span class="review-answer">✅ 正确答案: {{ q.answer }}</span>
                <span v-if="q.explanation" class="review-explanation">💡 {{ q.explanation }}</span>
              </div>
              <div class="review-actions">
                <button @click="reviewQuestion(q.id, 1)" class="btn-mini success">✓ 通过</button>
                <button @click="reviewQuestion(q.id, 2)" class="btn-mini danger">✗ 拒绝</button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div v-else class="exam-layout">
        <aside class="sidebar-card">
          <div class="card-header">题库导航</div>
          <div class="card-content">
            <div v-for="group in sidebarGroups" :key="group.meta.id" class="nav-group">
              <div class="group-label" :style="{ color: group.meta.color }" @click="toggleCategory(group.meta.id)">
                <span class="group-icon">{{ group.meta.icon }}</span>
                <span class="group-name">{{ group.meta.name }}</span>
                <span class="collapse-icon">{{ isCategoryCollapsed(group.meta.id) ? '▶' : '▼' }}</span>
              </div>
              <div v-show="!isCategoryCollapsed(group.meta.id)" class="group-items">
                <div v-for="(q, idx) in group.list" :key="q.id" @click="enterQuestion(q)"
                     :class="['nav-item', { active: currentQuestion && currentQuestion.id === q.id }]"
                     :style="currentQuestion && currentQuestion.id === q.id ? { backgroundColor: group.meta.color } : {}">
                  {{ idx + 1 }}
                </div>
              </div>
            </div>
          </div>
        </aside>

        <main class="paper-area">
          <div v-if="!currentQuestion" class="empty-state">👈 请选择题目</div>
          <div v-else class="question-container">
            <div class="tag-row"><span class="type-tag" :style="{ backgroundColor: currentCategoryInfo.bg, color: currentCategoryInfo.color }">{{ currentCategoryInfo.name }}</span></div>
            <h2 class="q-title">{{ currentQuestion.title }}</h2>
            <div class="options-list">
              <div v-for="opt in currentQuestion.options" :key="opt" class="option-card" :class="{'selected': selectedOption===opt}" @click="!showResult && (selectedOption = opt)"
                   :style="selectedOption===opt ? { borderColor: currentCategoryInfo.color, backgroundColor: currentCategoryInfo.bg } : {}">
                <div class="radio-circle" :style="selectedOption===opt ? { backgroundColor: currentCategoryInfo.color } : {}"></div>
                {{ opt }}
              </div>
            </div>
            <div class="action-area"><button v-if="!showResult" @click="submitAnswer" class="submit-btn" :style="{ backgroundColor: currentCategoryInfo.color }">提交 ✨</button></div>
            <div v-if="showResult" class="result-card" :class="selectedOption===currentQuestion.answer?'res-correct':'res-wrong'">
              <div class="res-icon">{{ selectedOption===currentQuestion.answer?'🎉':'🤔' }}</div>
              <div><h4>{{ selectedOption===currentQuestion.answer?'Bingo!':'Oops!' }}</h4><p>{{ currentQuestion.explain }}</p></div>
            </div>
            
            <div class="discussion-section">
              <div class="section-title">💬 讨论区</div>
              <div class="input-box">
                <input v-model="myComment" placeholder="输入想法..." @keyup.enter="submitComment" />
                <button @click="submitComment" :style="{ color: currentCategoryInfo.color }">发布</button>
              </div>
              <div class="comment-list">
                <div v-for="c in commentList" :key="c.id" class="comment-item">
                  <div class="c-avatar">{{ c.username ? c.username[0] : '?' }}</div>
                  <div class="c-main">
                    <div class="c-user">
                      <span>{{ c.username }}</span>
                      
                      <span class="c-time">
                        {{ c.createTime ? c.createTime.replace('T', ' ').substring(0, 16) : '刚刚' }}
                      </span>
                    </div>
                    <div class="c-txt">{{ c.content }}</div>
                    <div class="c-actions">
                      <span @click="handleAction(c, 1)" class="action-btn">
                        ❤️ {{ c.likeCount || 0 }}
                      </span>
                      
                      <span @click="handleAction(c, 2)" class="action-btn">
                        👎 {{ c.dislikeCount || 0 }}
                      </span>

                      
                      <span v-if="currentUser.id !== c.userId" @click="handleAction(c, 3)" class="action-btn report-btn">
                        🚨 举报
                      </span>
                      <span v-if="currentUser.id === c.userId || currentUser.role === 'ADMIN'" 
                            @click="deleteComment(c)" 
                            class="action-btn delete-btn">
                        🗑️ 删除
                      </span>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </main>
      </div>
    </div>

    <!-- 上传题目弹窗 -->
    <div v-if="showUploadPanel" class="modal-mask" @click.self="showUploadPanel = false">
      <div class="modal-box upload-modal">
        <h3>📝 上传新题目</h3>
        <div class="upload-form">
          <div class="form-group">
            <label>题目分类</label>
            <select v-model="uploadForm.categoryId" class="form-select">
              <option v-for="cat in categories" :key="cat.id" :value="cat.id">
                {{ cat.icon }} {{ cat.name }}
              </option>
            </select>
          </div>

          <div class="form-group">
            <label>题目内容</label>
            <textarea v-model="uploadForm.title" class="form-textarea" placeholder="请输入题目内容..." rows="3"></textarea>
          </div>

          <div class="form-group">
            <label>选项设置</label>
            <div v-for="(opt, idx) in uploadForm.options" :key="idx" class="option-input-group">
              <span class="option-label">{{ String.fromCharCode(65 + idx) }}.</span>
              <input v-model="uploadForm.options[idx]" class="form-input" placeholder="选项内容" />
              <button v-if="uploadForm.options.length > 2" @click="removeOption(idx)" class="btn-remove">✕</button>
            </div>
            <button v-if="uploadForm.options.length < 8" @click="addOption" class="btn-add-option">+ 添加选项</button>
          </div>

          <div class="form-group">
            <label>正确答案</label>
            <input v-model="uploadForm.answer" class="form-input" placeholder="如：A" />
          </div>

          <div class="form-group">
            <label>题目解析（可选）</label>
            <textarea v-model="uploadForm.explanation" class="form-textarea" placeholder="请输入题目解析..." rows="2"></textarea>
          </div>

          <div class="form-actions">
            <button @click="showUploadPanel = false" class="btn-cancel">取消</button>
            <button @click="uploadQuestion" class="btn-submit">提交审核</button>
          </div>
        </div>
      </div>
    </div>

    <!-- 我的上传记录弹窗 -->
    <div v-if="showMyUploadsPanel" class="modal-mask" @click.self="showMyUploadsPanel = false">
      <div class="modal-box uploads-modal">
        <h3>📂 我的上传记录</h3>
        <div v-if="myUploads.length === 0" class="empty-state">暂无上传记录</div>
        <div v-else class="uploads-list">
          <div v-for="q in myUploads" :key="q.id" class="upload-item">
            <div class="upload-header">
              <span class="upload-type">{{ getCategoryById(q.type).icon }} {{ getCategoryById(q.type).name }}</span>
              <span :class="['status-badge', getReviewStatusClass(q.reviewStatus)]">
                {{ getReviewStatusText(q.reviewStatus) }}
              </span>
            </div>
            <div class="upload-title">{{ q.title }}</div>
            <div class="upload-meta">
              <span>答案: {{ q.answer }}</span>
              <span class="upload-id">#{{ q.id }}</span>
            </div>
          </div>
        </div>
        <button @click="showMyUploadsPanel = false" class="btn-close-modal">关闭</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Nunito:wght@400;700&display=swap');
.app-container { font-family: 'Nunito', "PingFang SC", sans-serif; background: #f4f7f6; min-height: 100vh; display: flex; flex-direction: column; color: #444; }
.navbar { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); color: white; padding: 0 40px; height: 64px; display: flex; justify-content: space-between; align-items: center; box-shadow: 0 4px 20px rgba(118, 75, 162, 0.2); }
.brand { font-size: 20px; font-weight: 800; display: flex; align-items: center; gap: 8px; }
.sub-brand { font-size: 12px; background: rgba(255,255,255,0.2); padding: 2px 8px; border-radius: 20px; font-weight: normal; margin-left: 8px; }
.user-info { display: flex; gap: 15px; align-items: center; }
.avatar-circle { width: 32px; height: 32px; background: white; color: #764ba2; border-radius: 50%; text-align: center; line-height: 32px; font-weight: bold; }
.u-details { display: flex; flex-direction: column; text-align: right; line-height: 1.2; }
.u-name { font-size: 14px; font-weight: bold; }
.u-role { font-size: 10px; opacity: 0.8; }
.nav-btn { background: rgba(255,255,255,0.2); border: none; color: white; padding: 6px 16px; border-radius: 20px; cursor: pointer; font-size: 13px; transition: 0.2s; }
.nav-btn:hover { background: rgba(255,255,255,0.3); transform: translateY(-1px); }
.nav-btn.ghost { background: transparent; border: 1px solid rgba(255,255,255,0.4); }

/* 登录/注册卡片样式 */
.login-wrapper { flex: 1; display: flex; justify-content: center; align-items: center; background: #eef2f3; }
.login-card { background: white; padding: 40px 50px; width: 400px; border-radius: 24px; box-shadow: 0 10px 40px rgba(0,0,0,0.08); text-align: center; transition: 0.3s; }
.login-header h1 { margin: 0 0 10px 0; color: #333; font-size: 24px; }
.login-header p { color: #888; margin-bottom: 30px; font-size: 14px; }
.input-group input { width: 100%; padding: 15px; margin-bottom: 15px; border: 2px solid #f0f0f0; border-radius: 12px; background: #f9f9f9; outline: none; transition: 0.2s; box-sizing: border-box; }
.input-group input:focus { border-color: #764ba2; background: white; }

/* 按钮样式 */
.login-btn { width: 100%; padding: 15px; background: #764ba2; color: white; border: none; border-radius: 12px; font-size: 16px; font-weight: bold; cursor: pointer; box-shadow: 0 4px 15px rgba(118, 75, 162, 0.3); transition: 0.2s; }
.login-btn:hover { transform: translateY(-2px); }
.login-btn.register-color { background: #2ecc71; box-shadow: 0 4px 15px rgba(46, 204, 113, 0.3); } /* 注册按钮用绿色 */

/* 切换模式链接 */
.toggle-mode { margin-top: 20px; font-size: 14px; color: #666; }
.toggle-mode a { color: #764ba2; cursor: pointer; font-weight: bold; text-decoration: underline; margin-left: 5px; }

/* 布局与通用 */
.exam-layout { display: flex; gap: 30px; padding: 30px 40px; max-width: 1200px; margin: 0 auto; width: 100%; align-items: flex-start; }
.sidebar-card { width: 280px; background: white; border-radius: 20px; padding: 20px; box-shadow: 0 4px 20px rgba(0,0,0,0.03); position: sticky; top: 20px; }
.card-header { font-weight: bold; margin-bottom: 20px; color: #333; }
.nav-group { margin-bottom: 15px; }
.group-label {
  font-size: 13px;
  font-weight: bold;
  margin-bottom: 8px;
  padding-left: 5px;
  padding-right: 8px;
  padding-top: 6px;
  padding-bottom: 6px;
  cursor: pointer;
  user-select: none;
  display: flex;
  align-items: center;
  gap: 6px;
  border-radius: 8px;
  transition: background 0.2s;
}
.group-label:hover {
  background: #f5f5f5;
}
.group-icon {
  font-size: 14px;
}
.group-name {
  flex: 1;
}
.collapse-icon {
  font-size: 10px;
  color: #999;
  transition: transform 0.2s;
}
.group-items { display: flex; flex-wrap: wrap; gap: 8px; }
.nav-item { width: 36px; height: 36px; display: flex; justify-content: center; align-items: center; border-radius: 12px; background: #f8f9fa; color: #666; cursor: pointer; font-weight: bold; font-size: 14px; transition: 0.2s; }
.nav-item:hover, .nav-item.active { transform: scale(1.1); color: white; }

.paper-area { flex: 1; background: white; border-radius: 24px; padding: 40px; min-height: 600px; box-shadow: 0 4px 20px rgba(0,0,0,0.03); }
.empty-state { text-align: center; color: #ccc; margin-top: 100px; font-size: 18px; }
.type-tag { padding: 6px 12px; border-radius: 8px; font-size: 13px; font-weight: bold; display: inline-block; margin-bottom: 20px; }
.q-title { margin: 0 0 30px 0; color: #2d3436; line-height: 1.5; }
.options-list { display: flex; flex-direction: column; gap: 15px; }
.option-card { display: flex; align-items: center; padding: 18px; border: 2px solid #f0f0f0; border-radius: 16px; cursor: pointer; transition: 0.2s; background: #fff; }
.option-card:hover { border-color: #dcdcdc; background: #fafafa; }
.option-card.selected { border-width: 2px; font-weight: bold; }
.radio-circle { width: 20px; height: 20px; border: 2px solid #ddd; border-radius: 50%; margin-right: 15px; flex-shrink: 0; }
.selected .radio-circle { border: none; }
.action-area { margin-top: 30px; display: flex; justify-content: flex-end; }
.submit-btn { padding: 12px 30px; color: white; border: none; border-radius: 12px; font-size: 16px; font-weight: bold; cursor: pointer; box-shadow: 0 4px 12px rgba(0,0,0,0.1); }
.result-card { margin-top: 30px; padding: 20px; border-radius: 16px; display: flex; gap: 15px; animation: slideUp 0.3s ease; }
.res-correct { background: #edfff5; border: 1px solid #c3e6cb; color: #155724; }
.res-wrong { background: #fff5f5; border: 1px solid #f5c6cb; color: #721c24; }
.res-icon { font-size: 30px; }

/* 讨论区 */
.discussion-section { margin-top: 60px; border-top: 2px dashed #f0f0f0; padding-top: 30px; }
.section-title { font-weight: 800; font-size: 18px; margin-bottom: 20px; color: #333; }
.input-box { display: flex; gap: 10px; background: #f9f9f9; padding: 10px; border-radius: 12px; border: 1px solid #eee; }
.input-box input { flex: 1; border: none; background: transparent; outline: none; font-size: 14px; }
.input-box button { background: white; border: 1px solid #eee; padding: 8px 16px; border-radius: 8px; font-weight: bold; cursor: pointer; }
.comment-list { margin-top: 30px; display: flex; flex-direction: column; gap: 20px; }
.comment-item { display: flex; gap: 12px; }
.c-avatar { width: 36px; height: 36px; background: #e0e0e0; border-radius: 50%; color: white; display: flex; justify-content: center; align-items: center; font-weight: bold; font-size: 14px; }
.c-main { flex: 1; background: #f8f9fa; padding: 12px 16px; border-radius: 0 16px 16px 16px; }
.c-user { font-size: 13px; font-weight: bold; margin-bottom: 4px; color: #555; display: flex; justify-content: space-between; }
.c-txt { font-size: 14px; color: #333; line-height: 1.4; }
/* 找到 c-actions，确保它支持横向排列 */
.c-actions {
  margin-top: 8px;
  font-size: 12px;
  color: #888;
  display: flex;
  gap: 15px; /* 让按钮之间有间距 */
  cursor: pointer;
  align-items: center; /* 垂直居中 */
}

/* 按钮通用样式 */
.action-btn:hover {
  color: #764ba2; /* 悬停变色 */
  transform: scale(1.1); /* 微微放大 */
  transition: 0.2s;
  display: inline-block;
}

/* 举报按钮稍微放右边一点，或者变红 */
.report-btn {
  margin-left: auto; /* 把举报按钮推到最右边 */
  color: #e74c3c; /* 红色警告色 */
  opacity: 0.6;
}
.report-btn:hover {
  opacity: 1;
}

/* 后台表格 */
.admin-panel { background: white; padding: 30px; border-radius: 20px; max-width: 900px; margin: 30px auto; box-shadow: 0 5px 20px rgba(0,0,0,0.05); }
.data-table { width: 100%; border-collapse: separate; border-spacing: 0 10px; }
.data-table th { text-align: left; color: #999; font-size: 12px; padding: 0 15px; }
.data-table td { background: #fcfcfc; padding: 15px; border-top: 1px solid #f0f0f0; border-bottom: 1px solid #f0f0f0; }
.data-table td:first-child { border-left: 1px solid #f0f0f0; border-radius: 10px 0 0 10px; }
.data-table td:last-child { border-right: 1px solid #f0f0f0; border-radius: 0 10px 10px 0; }
.tag-report { background: #ffebeb; color: #e17055; padding: 2px 8px; border-radius: 4px; font-size: 12px; font-weight: bold; }
.status-dot { display: inline-block; width: 8px; height: 8px; border-radius: 50%; margin-right: 5px; }
.btn-mini { border: none; padding: 5px 12px; border-radius: 6px; cursor: pointer; font-size: 12px; margin-right: 5px; }
.btn-mini.danger { background: #ffecec; color: #d63031; }
.btn-mini.success { background: #e3fcef; color: #00b894; }
.modal-mask { position: fixed; inset: 0; background: rgba(0,0,0,0.5); display: flex; justify-content: center; align-items: center; }
.modal-box { background: white; padding: 20px; width: 400px; border-radius: 8px; max-height: 80vh; overflow-y: auto; }
.audit-item { border-bottom: 1px solid #eee; padding: 10px 0; }

@keyframes slideUp { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
/* 审计弹窗里的单条记录 */
.audit-item {
  border-bottom: 1px solid #eee;
  padding: 10px 0;
  display: flex;
  flex-direction: column; /* 垂直排列 */
  gap: 5px; /* 内容和信息的间距 */
}

/* 评论内容 */
.audit-content {
  color: #333;
  font-size: 14px;
}

/* 时间和地点 - 灰色小字 */
.audit-info {
  font-size: 12px;
  color: #999;
}

/* 删除按钮样式 */
.delete-btn {
  color: #95a5a6; /* 默认灰色 */
  font-size: 12px;
}
.delete-btn:hover {
  color: #e74c3c; /* 悬停变红 */
  font-weight: bold;
}
/* 时间样式 */
.c-time {
  font-size: 12px;
  color: #ccc;      /* 浅灰色 */
  font-weight: normal; /* 取消粗体 */
}

/* ================= 新增样式：管理员面板标签 ================= */
.admin-tabs {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
  border-bottom: 2px solid #f0f0f0;
  padding-bottom: 10px;
}

.tab-btn {
  padding: 10px 20px;
  border: none;
  background: transparent;
  color: #666;
  cursor: pointer;
  font-weight: bold;
  border-radius: 8px;
  transition: all 0.2s;
}

.tab-btn.active {
  background: #764ba2;
  color: white;
}

.tab-btn:hover:not(.active) {
  background: #f0f0f0;
}

/* ================= 新增样式：题目审核列表 ================= */
.review-list {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.review-item {
  background: #f9f9f9;
  padding: 20px;
  border-radius: 12px;
  border: 1px solid #eee;
}

.review-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.review-type {
  background: #e3f2fd;
  color: #1976d2;
  padding: 4px 12px;
  border-radius: 6px;
  font-size: 12px;
  font-weight: bold;
}

.review-id {
  color: #999;
  font-size: 12px;
}

.review-title {
  font-size: 16px;
  font-weight: bold;
  color: #333;
  margin-bottom: 15px;
  line-height: 1.5;
}

.review-options {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 15px;
}

.review-option {
  padding: 8px 12px;
  background: white;
  border-radius: 6px;
  border: 1px solid #eee;
  font-size: 14px;
}

.review-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
  margin-bottom: 15px;
  font-size: 14px;
}

.review-answer {
  color: #27ae60;
  font-weight: bold;
}

.review-explanation {
  color: #7f8c8d;
}

.review-actions {
  display: flex;
  gap: 10px;
}

/* ================= 新增样式：上传题目弹窗 ================= */
.upload-modal {
  width: 500px;
  max-height: 80vh;
  overflow-y: auto;
}

.upload-form {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.form-group label {
  font-weight: bold;
  color: #333;
  font-size: 14px;
}

.form-select,
.form-input,
.form-textarea {
  padding: 10px 12px;
  border: 1px solid #ddd;
  border-radius: 8px;
  font-size: 14px;
  outline: none;
  transition: border-color 0.2s;
}

.form-select:focus,
.form-input:focus,
.form-textarea:focus {
  border-color: #764ba2;
}

.form-textarea {
  resize: vertical;
  font-family: inherit;
}

.option-input-group {
  display: flex;
  align-items: center;
  gap: 10px;
}

.option-label {
  font-weight: bold;
  color: #764ba2;
  width: 25px;
}

.option-input-group .form-input {
  flex: 1;
}

.btn-remove {
  width: 24px;
  height: 24px;
  border: none;
  background: #ffecec;
  color: #e74c3c;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.btn-remove:hover {
  background: #ffd6d6;
}

.btn-add-option {
  padding: 8px 12px;
  border: 1px dashed #764ba2;
  background: transparent;
  color: #764ba2;
  border-radius: 6px;
  cursor: pointer;
  font-size: 13px;
  align-self: flex-start;
}

.btn-add-option:hover {
  background: #f3e5f5;
}

.form-actions {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
  margin-top: 10px;
}

.btn-cancel {
  padding: 10px 20px;
  border: 1px solid #ddd;
  background: white;
  color: #666;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
  font-weight: bold;
}

.btn-cancel:hover {
  background: #f5f5f5;
}

.btn-submit {
  padding: 10px 20px;
  border: none;
  background: #764ba2;
  color: white;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
  font-weight: bold;
}

.btn-submit:hover {
  background: #6b3aa0;
}

/* ================= 新增样式：我的上传记录弹窗 ================= */
.uploads-modal {
  width: 500px;
  max-height: 80vh;
  overflow-y: auto;
}

.uploads-list {
  display: flex;
  flex-direction: column;
  gap: 15px;
  max-height: 60vh;
  overflow-y: auto;
}

.upload-item {
  background: #f9f9f9;
  padding: 15px;
  border-radius: 10px;
  border: 1px solid #eee;
}

.upload-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.upload-type {
  background: #e3f2fd;
  color: #1976d2;
  padding: 3px 10px;
  border-radius: 5px;
  font-size: 12px;
  font-weight: bold;
}

.status-badge {
  padding: 3px 10px;
  border-radius: 5px;
  font-size: 12px;
  font-weight: bold;
}

.status-pending {
  background: #fff3cd;
  color: #856404;
}

.status-approved {
  background: #d4edda;
  color: #155724;
}

.status-rejected {
  background: #f8d7da;
  color: #721c24;
}

.upload-title {
  font-size: 14px;
  font-weight: bold;
  color: #333;
  margin-bottom: 8px;
  line-height: 1.4;
}

.upload-meta {
  display: flex;
  justify-content: space-between;
  font-size: 12px;
  color: #999;
}

.upload-id {
  color: #bbb;
}

.btn-close-modal {
  width: 100%;
  padding: 12px;
  border: none;
  background: #f5f5f5;
  color: #666;
  border-radius: 8px;
  cursor: pointer;
  font-weight: bold;
  margin-top: 20px;
}

.btn-close-modal:hover {
  background: #eee;
}
</style>