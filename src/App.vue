<script setup>
import { ref, reactive, computed } from 'vue'

// ================= 0. 配置区 =================
// 确保这里指向你的 SpringBoot 端口
const API_BASE = 'http://localhost:8080/api'

// 分类配置 (保持 Eazy 风格)
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
const questions = ref([
  { id: 101, type: 'politics', title: '党的二十大报告指出，中国式现代化的本质要求是？', options: ['A. 坚持中国共产党领导', 'B. 坚持改革开放', 'C. 丰富人民精神世界', 'D. 实现高质量发展'], answer: 'A. 坚持中国共产党领导', explain: '💡 知识点：二十大报告明确指出，中国式现代化的本质要求是：坚持中国共产党领导...' },
  { id: 201, type: 'common', title: '关于“天宫一号”，下面哪个说法是对的？', options: ['A. 中国第一个空间实验室', 'B. 载人飞船', 'C. 气象卫星', 'D. 探月工程'], answer: 'A. 中国第一个空间实验室', explain: '💡 冷知识：天宫一号是中国第一个目标飞行器和空间实验室，不是飞船哦。' },
  { id: 301, type: 'verbal', title: '填个词让句子通顺：“历史是最好的教科书，我们____历史，是为了总结经验。”', options: ['A. 学习', 'B. 借鉴', 'C. 回顾', 'D. 研究'], answer: 'A. 学习', explain: '💡 思路：搭配“历史”且后文提到了“总结经验”，用“学习”最自然。' },
  { id: 401, type: 'logic', title: '类比推理：医生之于患者，好比教师之于？', options: ['A. 学校', 'B. 学生', 'C. 教材', 'D. 教室'], answer: 'B. 学生', explain: '💡 逻辑链：职业 vs 服务对象的关系。' },
  { id: 501, type: 'math', title: '甲乙两地相距100公里，A车60km/h，B车40km/h，相向而行多久相遇？', options: ['A. 30分钟', 'B. 1小时', 'C. 1.5小时', 'D. 2小时'], answer: 'B. 1小时', explain: '💡 算式：时间 = 路程 / (速度A + 速度B) = 100 / 100 = 1。' },
  { id: 601, type: 'data', title: '去年GDP是10万亿，今年增长5.2%，算算今年是多少？', options: ['A. 10.52万亿', 'B. 10.5万亿', 'C. 11.2万亿', 'D. 10.2万亿'], answer: 'A. 10.52万亿', explain: '💡 速算：10 × (1 + 0.052) = 10.52，口算就能出来！' }
])

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
</script>

<template>
  <div class="app-container">
    
    <header class="navbar">
      <div class="brand">
        <span class="logo-icon">⚡</span> EazyExam <span class="sub-brand">轻松刷题社区</span>
      </div>
      <div v-if="isLoggedIn" class="user-info">
        <div class="avatar-circle">{{ currentUser.username ? currentUser.username[0].toUpperCase() : 'U' }}</div>
        <div class="u-details">
          <span class="u-name">{{ currentUser.username }}</span>
          <span class="u-role">{{ currentUser.role === 'ADMIN' ? 'Admin' : 'User' }}</span>
        </div>
        <button v-if="currentUser.role === 'ADMIN'" @click="showAdminPanel = !showAdminPanel" class="nav-btn">{{ showAdminPanel ? '刷题去' : '管后台' }}</button>
        <button @click="handleLogout" class="nav-btn ghost">退出</button>
      </div>
    </header>

    <div v-if="!isLoggedIn" class="login-wrapper">
      <div class="login-card">
        <div class="login-header">
          <h1>{{ isRegisterMode ? '🚀 创建新账号' : '👋 Hi, Welcome!' }}</h1>
          <p>{{ isRegisterMode ? '加入 EazyExam，开启挑战' : '登录你的账号继续刷题' }}</p>
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

      <div v-else class="exam-layout">
        <aside class="sidebar-card">
          <div class="card-header">题库导航</div>
          <div class="card-content">
            <div v-for="group in sidebarGroups" :key="group.meta.id" class="nav-group">
              <div class="group-label" :style="{ color: group.meta.color }">{{ group.meta.icon }} {{ group.meta.name }}</div>
              <div class="group-items">
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
.group-label { font-size: 13px; font-weight: bold; margin-bottom: 8px; padding-left: 5px; opacity: 0.8; }
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
</style>