<script setup>
import { ref, computed } from 'vue'

// ==========================================
// 1. 全局状态与配置
// ==========================================
const API_BASE = 'http://localhost:8080/api' // 后端地址

// 用户状态
const isLoggedIn = ref(false)
const isRegisterMode = ref(false) // 是否处于注册模式
const loginLoading = ref(false)
const username = ref('')
const password = ref('')
const nickname = ref('')

// 角色与权限
const currentUserRole = ref('USER') // USER 或 ADMIN
const showAdminPanel = ref(false)   // 是否显示管理员面板
const userList = ref([])            // 管理员看到的用户列表

// 题库状态
const currentCategory = ref(null)
const currentQuestion = ref(null)
const selectedOption = ref(null)
const hasSubmitted = ref(false)
const submitResult = ref(null)

//评论状态
const currentUserId = ref(null) // 关键：存用户ID
const commentList = ref([])
const myComment = ref('')


// ==========================================
// 2. 题库数据 (保持不变)
// ==========================================
const categories = [
  { id: 'politics', name: '政治理论', icon: '🚩', color: '#c0392b', bg: '#fadbd8' },
  { id: 'common',   name: '常识判断', icon: '🌍', color: '#e67e22', bg: '#fdebd0' },
  { id: 'verbal',   name: '言语理解', icon: '📝', color: '#2980b9', bg: '#d6eaf8' },
  { id: 'logic',    name: '判断推理', icon: '🧩', color: '#8e44ad', bg: '#ebdef0' },
  { id: 'math',     name: '数量关系', icon: '📐', color: '#27ae60', bg: '#d5f5e3' },
  { id: 'data',     name: '资料分析', icon: '📊', color: '#2c3e50', bg: '#d6dbdf' }
]

const questions = ref([
  { id: 101, categoryId: 'politics', title: '关于新发展理念...', description: "下列不属于新发展理念内容的是：", options: [{id:'A', text:'创新'}, {id:'B', text:'协调'}, {id:'C', text:'高速'}, {id:'D', text:'共享'}], correct: 'C', explanation: '新发展理念包括：创新、协调、绿色、开放、共享。', isSolved: false },
  { id: 201, categoryId: 'common', title: '二十四节气排序...', description: "下列节气按时间先后排序正确的是：", options: [{id:'A', text:'立春、惊蛰、雨水'}, {id:'B', text:'立夏、小满、芒种'}, {id:'C', text:'白露、秋分、寒露'}, {id:'D', text:'冬至、大雪、小寒'}], correct: 'B', explanation: 'B项正确。A应为立春、雨水、惊蛰。', isSolved: false },
  { id: 301, categoryId: 'verbal', title: '成语辨析...', description: "填入划横线部分最恰当的一项是：他在工作中一向______，深得领导赏识。", options: [{id:'A', text:'兢兢业业'}, {id:'B', text:'随波逐流'}, {id:'C', text:'好高骛远'}, {id:'D', text:'敷衍塞责'}], correct: 'A', explanation: '褒义词语境。', isSolved: false },
  { id: 401, categoryId: 'logic', title: '图形推理...', description: "把下面的六个图形分为两类，使每一类图形都有各自的共同特征或规律。", options: [{id:'A', text:'①③④，②⑤⑥'}, {id:'B', text:'①③⑥，②④⑤'}, {id:'C', text:'①②③，④⑤⑥'}, {id:'D', text:'①④⑤，②③⑥'}], correct: 'B', explanation: '考察轴对称与中心对称。', isSolved: false },
  { id: 501, categoryId: 'math', title: '行程问题...', description: "甲乙两人同时从A地出发前往B地，甲每分钟走80米，乙每分钟走60米...", options: [{id:'A', text:'1200米'}, {id:'B', text:'1440米'}, {id:'C', text:'1600米'}, {id:'D', text:'1800米'}], correct: 'B', explanation: '简单的追及问题计算。', isSolved: false },
  { id: 601, categoryId: 'data', title: '2023年GDP增长率...', material: "2023年全年国内生产总值1260582亿元...", description: "根据资料，2023年国内生产总值比上年增长了多少？", options: [{id:'A', text:'4.5%'}, {id:'B', text:'5.0%'}, {id:'C', text:'5.2%'}, {id:'D', text:'5.5%'}], correct: 'C', explanation: '直接查找资料第一句即可得出。', isSolved: false }
])

// ==========================================
// 3. 核心业务逻辑 (登录、注册、管理)
// ==========================================

// 处理登录和注册
async function handleAuth() {
  if (!username.value || !password.value) {
    alert('请输入账号和密码')
    return
  }
  
  loginLoading.value = true
  
  try {
    const url = isRegisterMode.value ? `${API_BASE}/register` : `${API_BASE}/login`
    
    // 发送请求
    const res = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        username: username.value,
        password: password.value,
        nickname: isRegisterMode.value ? '新用户' : undefined
      })
    })

    const data = await res.json()

    // 逻辑判定
    if (data.code === 200) {
      currentUserId.value = data.data.id
      if (isRegisterMode.value) {
        // 注册成功 -> 切换回登录
        alert('注册成功，请直接登录！')
        isRegisterMode.value = false
      } else {
        // 登录成功 -> 进入系统
        isLoggedIn.value = true
        currentUserRole.value = data.data.role
        nickname.value = data.data.nickname
        
        // 如果是管理员，直接开启后台面板并加载数据
        if (currentUserRole.value === 'ADMIN') {
          showAdminPanel.value = true
          fetchUserList()
        }
      }
    } else {
      alert(data.msg || '操作失败')
    }

  } catch (err) {
    console.error(err)
    alert('连接服务器失败，请确认后端已启动')
  } finally {
    loginLoading.value = false
  }
}

// 退出登录
function handleLogout() {
  isLoggedIn.value = false
  username.value = ''
  password.value = ''
  showAdminPanel.value = false
  currentCategory.value = null
  currentQuestion.value = null
}

// === 管理员功能 ===
async function fetchUserList() {
  try {
    const res = await fetch(`${API_BASE}/user/list`)
    const data = await res.json()
    if (data.code === 200) {
      userList.value = data.data
    }
  } catch (e) {
    alert('获取用户列表失败')
  }
}

async function handleDeleteUser(id) {
  if (!confirm('确定要删除这个用户吗？')) return
  try {
    const res = await fetch(`${API_BASE}/user/delete/${id}`, { method: 'DELETE' })
    const data = await res.json()
    if (data.code === 200) {
      alert('删除成功')
      fetchUserList() // 刷新列表
    } else {
      alert(data.msg)
    }
  } catch (e) {
    alert('删除请求失败')
  }
}

// ==========================================
// 4. 答题交互逻辑
// ==========================================
const filteredQuestions = computed(() => {
  if (!currentCategory.value) return []
  return questions.value.filter(q => q.categoryId === currentCategory.value.id)
})

function getCountByCategory(catId) {
  return questions.value.filter(q => q.categoryId === catId).length
}

function selectCategory(cat) {
  currentCategory.value = cat
}

function enterQuestion(q) {
  currentQuestion.value = q
  selectedOption.value = null
  hasSubmitted.value = false
  submitResult.value = null

  //加载评论
  commentList.value = []
  fetchComments(q.id)
}

function selectOption(id) {
  if (!hasSubmitted.value) selectedOption.value = id
}

function submitAnswer() {
  hasSubmitted.value = true
  if (selectedOption.value === currentQuestion.value.correct) {
    submitResult.value = { text: '✅ 回答正确', type: 'success' }
    currentQuestion.value.isSolved = true
  } else {
    submitResult.value = { text: '❌ 回答错误', type: 'error' }
  }
}

function getOptionClass(id) {
  if (!hasSubmitted.value) return selectedOption.value === id ? 'selected' : ''
  if (id === currentQuestion.value.correct) return 'correct'
  if (id === selectedOption.value) return 'wrong'
  return ''
}
// 获取评论函数
async function fetchComments(qid) {
  try {
    const res = await fetch(`${API_BASE}/comment/list?questionId=${qid}`)
    const data = await res.json()
    if (data.code === 200) commentList.value = data.data
  } catch (e) { console.error(e) }
}

//提交评论函数
async function submitComment() {
  if (!myComment.value.trim()) return alert('说点什么吧')
  if (!currentUserId.value) return alert('请先登录')

  const res = await fetch(`${API_BASE}/comment/add`, {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({
      userId: currentUserId.value,
      questionId: currentQuestion.value.id,
      content: myComment.value
    })
  })
  const data = await res.json()
  if (data.code === 200) {
    myComment.value = ''
    fetchComments(currentQuestion.value.id)
  } else {
    alert(data.msg)
  }
}

// 处理点赞/点踩
async function handleAction(comment, type) {
  if (!currentUserId.value) return alert('请先登录')
  if (type === 1) comment.likeCount = (comment.likeCount || 0) + 1
  if (type === 2) comment.dislikeCount = (comment.dislikeCount || 0) + 1

  const res = await fetch(`${API_BASE}/comment/action`, {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({
      userId: currentUserId.value,
      commentId: comment.id,
      type: type
    })
  })
  const data = await res.json()
  
  if (data.code !== 200) {
    alert(data.msg) // 如果重复点赞，报错
    // 回滚数字
    if (type === 1) comment.likeCount--
    if (type === 2) comment.dislikeCount--
  } else if (type === 3) {
    alert('举报成功')
  }
}
</script>

<template>
  <div class="app-container">
    
    <div v-if="!isLoggedIn" class="login-wrapper">
      <div class="login-card">
        <h1>{{ isRegisterMode ? '📝 注册账号' : '🎓 考试系统' }}</h1>
        <div class="form-item">
          <input v-model="username" type="text" placeholder="请输入用户名" />
        </div>
        <div class="form-item">
          <input v-model="password" type="password" placeholder="请输入密码" />
        </div>
        
        <button class="login-btn" @click="handleAuth" :disabled="loginLoading">
          {{ loginLoading ? '处理中...' : (isRegisterMode ? '立即注册' : '登录系统') }}
        </button>
        
        <div class="toggle-mode" @click="isRegisterMode = !isRegisterMode">
          {{ isRegisterMode ? '已有账号？去登录' : '没有账号？去注册' }}
        </div>
      </div>
    </div>

    <div v-else class="main-layout">
      
      <header class="navbar">
        <div class="logo">
          {{ showAdminPanel ? '🛡️ 后台管理系统' : '✍️ 在线刷题平台' }}
        </div>
        <div class="user-info">
          <span>欢迎, {{ nickname || username }} ({{ currentUserRole }})</span>
          <button @click="handleLogout" class="logout-btn">退出</button>
        </div>
      </header>

      <div v-if="showAdminPanel" class="admin-panel">
        <div class="panel-header">
          <h2>用户管理列表</h2>
          <button @click="fetchUserList" class="refresh-btn">刷新列表</button>
        </div>
        <table class="user-table">
          <thead>
            <tr>
              <th>ID</th>
              <th>用户名</th>
              <th>昵称</th>
              <th>角色</th>
              <th>操作</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="user in userList" :key="user.id">
              <td>{{ user.id }}</td>
              <td>{{ user.username }}</td>
              <td>{{ user.nickname }}</td>
              <td>
                <span :class="user.role === 'ADMIN' ? 'tag-admin' : 'tag-user'">
                  {{ user.role }}
                </span>
              </td>
              <td>
                <button 
                  v-if="user.role !== 'ADMIN'" 
                  @click="handleDeleteUser(user.id)" 
                  class="del-btn">
                  删除
                </button>
                <span v-else style="color:#999; font-size:12px;">不可操作</span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <div v-else class="student-view">
        
        <div v-if="!currentCategory" class="category-grid">
          <div 
            v-for="cat in categories" 
            :key="cat.id" 
            class="card"
            :style="{ '--card-color': cat.color, '--card-bg': cat.bg }"
            @click="selectCategory(cat)"
          >
            <div class="icon">{{ cat.icon }}</div>
            <h3>{{ cat.name }}</h3>
            <p>共 {{ getCountByCategory(cat.id) }} 题</p>
          </div>
        </div>

        <div v-else-if="!currentQuestion" class="question-list">
          <button @click="currentCategory = null" class="back-btn">⬅ 返回题库</button>
          <h2>{{ currentCategory.name }} - 题目列表</h2>
          <div class="list-wrapper">
            <div 
              v-for="(q, index) in filteredQuestions" 
              :key="q.id" 
              class="q-item" 
              @click="enterQuestion(q)"
            >
              <span class="q-status">{{ q.isSolved ? '✅' : '⬜' }}</span>
              <span>第 {{ index + 1 }} 题：{{ q.title }}</span>
            </div>
          </div>
        </div>

        <div v-else class="question-detail">
          <button @click="currentQuestion = null" class="back-btn">⬅ 返回列表</button>
          
          <div class="detail-card">
            <h3>{{ currentQuestion.description }}</h3>
            <p v-if="currentQuestion.material" class="material">{{ currentQuestion.material }}</p>
            
            <div class="options">
              <div 
                v-for="opt in currentQuestion.options" 
                :key="opt.id"
                class="option-item"
                :class="getOptionClass(opt.id)"
                @click="selectOption(opt.id)"
              >
                <span class="opt-label">{{ opt.id }}.</span> {{ opt.text }}
              </div>
            </div>

            <div class="actions">
              <button 
                v-if="!hasSubmitted" 
                @click="submitAnswer" 
                class="submit-btn" 
                :disabled="!selectedOption">
                提交答案
              </button>
              
              <div v-else class="result-area">
                <div :class="submitResult.type">{{ submitResult.text }}</div>
                <div class="explanation">解析：{{ currentQuestion.explanation }}</div>
              </div>
              <div class="comment-section">
                <h3>💬 互动讨论区</h3>

                <div class="input-area">
                  <textarea v-model="myComment" placeholder="发表你的高见..."></textarea>
                  <button @click="submitComment">发送</button>
                </div>

                <div class="comment-list">
                  <div v-if="commentList.length === 0" style="color:#999;text-align:center">暂无评论</div>
                  
                  <div v-for="c in commentList" :key="c.id" class="c-item">
                    <div class="c-head">
                      <span class="avatar">{{ c.nickname ? c.nickname[0] : '某' }}</span>
                      <span class="name">{{ c.nickname || '神秘考友' }}</span>
                      <span class="date">{{ c.createTime?.replace('T', ' ').slice(0,16) }}</span>
                    </div>
                    
                    <div class="c-content">{{ c.content }}</div>
                    
                    <div class="c-actions">
                      <span @click="handleAction(c, 1)" class="act-btn">👍 {{ c.likeCount || 0 }}</span>
                      <span @click="handleAction(c, 2)" class="act-btn">👎 {{ c.dislikeCount || 0 }}</span>
                      <span @click="handleAction(c, 3)" class="act-btn report">🚩 举报</span>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

    </div>
  </div>
</template>

<style scoped>
/* 全局容器 */
.app-container {
  font-family: 'Segoe UI', sans-serif;
  min-height: 100vh;
  background-color: #f4f6f8;
  color: #333;
}

/* 1. 登录页样式 */
.login-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}
.login-card {
  background: white;
  padding: 40px;
  border-radius: 12px;
  box-shadow: 0 10px 25px rgba(0,0,0,0.1);
  width: 350px;
  text-align: center;
}
.form-item { margin-bottom: 15px; }
.form-item input {
  width: 100%;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  box-sizing: border-box;
}
.login-btn {
  width: 100%;
  padding: 12px;
  background: #667eea;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 16px;
}
.login-btn:hover { background: #5a6fd1; }
.toggle-mode { margin-top: 15px; color: #667eea; cursor: pointer; font-size: 14px; }

/* 2. 主界面样式 */
.navbar {
  background: white;
  padding: 0 30px;
  height: 60px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  box-shadow: 0 2px 10px rgba(0,0,0,0.05);
}
.logo { font-size: 20px; font-weight: bold; color: #2c3e50; }
.logout-btn { 
  margin-left: 15px; 
  padding: 5px 15px; 
  border: 1px solid #ddd; 
  background: transparent; 
  cursor: pointer; 
  border-radius: 4px;
}

/* 3. 管理员表格样式 */
.admin-panel { padding: 30px; max-width: 900px; margin: 0 auto; }
.panel-header { display: flex; justify-content: space-between; margin-bottom: 20px; }
.refresh-btn { padding: 8px 16px; background: #3498db; color: white; border: none; border-radius: 4px; cursor: pointer; }
.user-table { width: 100%; border-collapse: collapse; background: white; border-radius: 8px; overflow: hidden; box-shadow: 0 2px 8px rgba(0,0,0,0.05); }
.user-table th, .user-table td { padding: 15px; text-align: left; border-bottom: 1px solid #eee; }
.user-table th { background: #f8f9fa; font-weight: 600; color: #555; }
.tag-admin { background: #ffebee; color: #c0392b; padding: 4px 8px; border-radius: 4px; font-size: 12px; }
.tag-user { background: #e8f5e9; color: #27ae60; padding: 4px 8px; border-radius: 4px; font-size: 12px; }
.del-btn { background: #ff4757; color: white; border: none; padding: 6px 12px; border-radius: 4px; cursor: pointer; }

/* 4. 题库卡片样式 */
.student-view { padding: 30px; max-width: 1000px; margin: 0 auto; }
.category-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 20px; }
.card {
  background: var(--card-bg);
  color: var(--card-color);
  padding: 25px;
  border-radius: 12px;
  cursor: pointer;
  transition: transform 0.2s;
  text-align: center;
}
.card:hover { transform: translateY(-5px); }
.card .icon { font-size: 40px; margin-bottom: 10px; }

/* 5. 题目列表 & 详情 */
.back-btn { margin-bottom: 20px; padding: 8px 15px; cursor: pointer; border: none; background: #e0e0e0; border-radius: 4px; }
.list-wrapper { background: white; padding: 20px; border-radius: 8px; }
.q-item { padding: 15px; border-bottom: 1px solid #eee; cursor: pointer; display: flex; gap: 10px; }
.q-item:hover { background: #f9f9f9; }

.detail-card { background: white; padding: 30px; border-radius: 12px; box-shadow: 0 2px 10px rgba(0,0,0,0.05); }
.material { background: #f8f9fa; padding: 15px; border-radius: 6px; color: #666; font-size: 14px; margin-bottom: 15px; }
.option-item {
  padding: 12px 15px;
  border: 2px solid #eee;
  margin-bottom: 10px;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
}
.option-item.selected { border-color: #3498db; background: #ebf5fb; }
.option-item.correct { border-color: #2ecc71; background: #eafaf1; }
.option-item.wrong { border-color: #e74c3c; background: #fdedec; }
.submit-btn { width: 100%; padding: 15px; background: #2c3e50; color: white; border: none; border-radius: 6px; font-size: 16px; cursor: pointer; margin-top: 20px; }
.submit-btn:disabled { background: #ccc; cursor: not-allowed; }
.result-area { margin-top: 20px; padding-top: 20px; border-top: 1px solid #eee; }
.explanation { margin-top: 10px; color: #666; font-size: 14px; line-height: 1.6; }
.success { color: #27ae60; font-weight: bold; font-size: 18px; }
.error { color: #c0392b; font-weight: bold; font-size: 18px; }
/* 评论区整体 */
.comment-section { margin-top: 40px; border-top: 1px solid #eee; padding-top: 20px; }

/* 输入区 */
.input-area textarea { width: 100%; height: 80px; padding: 10px; border: 1px solid #ddd; border-radius: 5px; }
.input-area button { margin-top: 10px; float: right; background: #3498db; color: white; border: none; padding: 5px 15px; border-radius: 4px; cursor: pointer; }

/* 列表项 */
.comment-list { margin-top: 50px; }
.c-item { background: #f8f9fa; padding: 15px; border-radius: 8px; margin-bottom: 15px; }
.c-head { display: flex; align-items: center; margin-bottom: 10px; font-size: 14px; color: #666; }
.avatar { width: 28px; height: 28px; background: #9b59b6; color: white; border-radius: 50%; text-align: center; line-height: 28px; margin-right: 8px; font-size: 12px; }
.name { font-weight: bold; margin-right: auto; }

/* 动作按钮栏 */
.c-actions { margin-top: 10px; display: flex; gap: 15px; font-size: 13px; color: #777; cursor: pointer; }
.act-btn:hover { color: #3498db; }
.report:hover { color: red; }
</style>