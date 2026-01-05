# 题目管理 API 接口文档

## 基础信息
- **基础URL**: `http://localhost:8080`
- **Content-Type**: `application/json`
- **所有接口支持跨域**: ✅

---

## 1. 获取题目列表（已通过审核）

### 接口信息
- **URL**: `/api/question/list`
- **方法**: `GET`
- **描述**: 获取所有已通过管理员审核的题目列表

### 请求示例
```bash
GET http://localhost:8080/api/question/list
```

### 响应示例
```json
{
  "code": 200,
  "msg": "获取成功",
  "data": [
    {
      "id": 1,
      "type": "选择题",
      "title": "Java中哪个关键字用于定义常量？",
      "options": "[\"A. const\", \"B. final\", \"C. static\", \"D. constant\"]",
      "answer": "B",
      "explanation": "final关键字用于定义常量",
      "reviewStatus": 1,
      "uploaderId": 1
    }
  ]
}
```

---

## 2. 用户上传题目

### 接口信息
- **URL**: `/api/question/upload`
- **方法**: `POST`
- **描述**: 用户上传新的题目，上传后默认为"待审核"状态，需要管理员审核后才能显示

### 请求参数
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| type | String | 是 | 题目类型（如：选择题、判断题、填空题等） |
| title | String | 是 | 题目内容 |
| options | String | 是 | 选项，JSON字符串格式，如："[\"A. 选项1\",\"B. 选项2\"]" |
| answer | String | 是 | 答案 |
| explanation | String | 否 | 题目解析/解释 |
| uploaderId | Integer | 是 | 上传者用户ID |

### 请求示例
```json
POST http://localhost:8080/api/question/upload
Content-Type: application/json

{
  "type": "选择题",
  "title": "以下哪个是Java的基本数据类型？",
  "options": "[\"A. String\", \"B. Integer\", \"C. int\", \"D. Object\"]",
  "answer": "C",
  "explanation": "int是Java的8种基本数据类型之一",
  "uploaderId": 1
}
```

### 响应示例
```json
{
  "code": 200,
  "msg": "题目上传成功，等待管理员审核"
}
```

### 错误响应
```json
{
  "code": 500,
  "msg": "上传题目失败: [错误详情]"
}
```

---

## 3. 获取待审核题目列表（管理员）

### 接口信息
- **URL**: `/api/question/pending`
- **方法**: `GET`
- **描述**: 管理员获取所有待审核的题目列表

### 请求示例
```bash
GET http://localhost:8080/api/question/pending
```

### 响应示例
```json
{
  "code": 200,
  "msg": "获取成功",
  "data": [
    {
      "id": 5,
      "type": "选择题",
      "title": "Python是什么类型的语言？",
      "options": "[\"A. 编译型\", \"B. 解释型\", \"C. 汇编型\", \"D. 机器语言\"]",
      "answer": "B",
      "explanation": "Python是解释型语言",
      "reviewStatus": 0,
      "uploaderId": 2
    }
  ]
}
```

---

## 4. 审核题目（管理员）

### 接口信息
- **URL**: `/api/question/review`
- **方法**: `POST`
- **描述**: 管理员审核题目（通过或拒绝）

### 请求参数
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | Integer | 是 | 题目ID |
| reviewStatus | Integer | 是 | 审核状态：1-通过，2-拒绝 |

### 请求示例
```bash
# 通过题目
POST http://localhost:8080/api/question/review?id=5&reviewStatus=1

# 拒绝题目
POST http://localhost:8080/api/question/review?id=5&reviewStatus=2
```

### 响应示例（通过）
```json
{
  "code": 200,
  "msg": "题目审核通过成功"
}
```

### 响应示例（拒绝）
```json
{
  "code": 200,
  "msg": "题目审核拒绝成功"
}
```

### 错误响应
```json
{
  "code": 400,
  "msg": "审核状态参数错误"
}
```

```json
{
  "code": 500,
  "msg": "审核失败，题目不存在"
}
```

---

## 5. 查看我的上传记录

### 接口信息
- **URL**: `/api/question/my`
- **方法**: `GET`
- **描述**: 查看指定用户上传的所有题目（包括待审核、已通过、已拒绝）

### 请求参数
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| uploaderId | Integer | 是 | 上传者用户ID |

### 请求示例
```bash
GET http://localhost:8080/api/question/my?uploaderId=1
```

### 响应示例
```json
{
  "code": 200,
  "msg": "获取成功",
  "data": [
    {
      "id": 1,
      "type": "选择题",
      "title": "Java中哪个关键字用于定义常量？",
      "options": "[\"A. const\", \"B. final\", \"C. static\", \"D. constant\"]",
      "answer": "B",
      "explanation": "final关键字用于定义常量",
      "reviewStatus": 1,
      "uploaderId": 1
    },
    {
      "id": 3,
      "type": "判断题",
      "title": "Java支持多继承",
      "options": "[]",
      "answer": "错误",
      "explanation": "Java类不支持多继承，但接口支持",
      "reviewStatus": 0,
      "uploaderId": 1
    }
  ]
}
```

---

## 审核状态说明

| 状态值 | 说明 | 含义 |
|--------|------|------|
| 0 | 待审核 | 题目已上传，等待管理员审核 |
| 1 | 已通过 | 题目审核通过，可以在题目列表中显示 |
| 2 | 已拒绝 | 题目审核未通过，不会显示在题目列表中 |

---

## 完整的前端集成示例

### 上传题目
```javascript
async function uploadQuestion(questionData) {
  try {
    const response = await fetch('http://localhost:8080/api/question/upload', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(questionData)
    });

    const result = await response.json();
    if (result.code === 200) {
      alert('题目上传成功，等待管理员审核');
    } else {
      alert('上传失败：' + result.msg);
    }
  } catch (error) {
    console.error('上传失败:', error);
  }
}

// 使用示例
uploadQuestion({
  type: "选择题",
  title: "你的题目",
  options: "[\"A. 选项1\", \"B. 选项2\", \"C. 选项3\", \"D. 选项4\"]",
  answer: "A",
  explanation: "题目解析",
  uploaderId: 1
});
```

### 获取题目列表
```javascript
async function getQuestionList() {
  try {
    const response = await fetch('http://localhost:8080/api/question/list');
    const result = await response.json();

    if (result.code === 200) {
      console.log('题目列表:', result.data);
      return result.data;
    }
  } catch (error) {
    console.error('获取失败:', error);
  }
}
```

### 管理员审核题目
```javascript
async function reviewQuestion(questionId, isApproved) {
  try {
    const reviewStatus = isApproved ? 1 : 2;
    const response = await fetch(
      `http://localhost:8080/api/question/review?id=${questionId}&reviewStatus=${reviewStatus}`,
      { method: 'POST' }
    );

    const result = await response.json();
    if (result.code === 200) {
      alert(result.msg);
    } else {
      alert('审核失败：' + result.msg);
    }
  } catch (error) {
    console.error('审核失败:', error);
  }
}
```

---

## 注意事项

1. **题目列表接口** 只返回审核状态为"已通过"(1)的题目
2. **上传题目后** 默认状态为"待审核"(0)，不会立即在题目列表中显示
3. **审核状态** 需要管理员通过审核接口修改
4. **options字段** 需要传JSON字符串格式，前端需要用 `JSON.stringify()` 处理
5. **uploaderId** 应该从登录用户信息中获取

---

## 数据库准备

在使用接口前，请先在数据库中执行以下SQL：

```sql
-- 添加审核相关字段
ALTER TABLE question ADD COLUMN review_status INT DEFAULT 0 COMMENT '审核状态: 0-待审核, 1-已通过, 2-已拒绝';
ALTER TABLE question ADD COLUMN uploader_id INT DEFAULT NULL COMMENT '上传者ID';

-- 将已有题目设置为已通过状态
UPDATE question SET review_status = 1 WHERE review_status IS NULL;
```

---

**更新时间**: 2025-01-05
**版本**: v1.0
