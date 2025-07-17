<script setup>
import { ref, onMounted } from 'vue'
import WaterTankProgress from './WaterTankProgress.vue'
import { bitable } from '@lark-base-open/js-sdk'
import {
  ElButton,
  ElForm,
  ElFormItem,
  ElInput,
  ElAlert,
  ElDivider,
  ElCard,
  ElDialog,
  ElTable,
  ElTableColumn,
  ElLink,
  ElMessage,
  ElSlider,
  ElInputNumber,
  ElPagination
} from 'element-plus'
import { CircleCheck, Close, Refresh } from '@element-plus/icons-vue'

const FieldType = {
  Text: 1,
  Number: 2,
  SingleSelect: 3,
  MultiSelect: 4,
  DateTime: 5,
  Checkbox: 7,
  User: 11,
  Phone: 13,
  Url: 15,
  Attachment: 17,
  SingleLink: 18,
  Lookup: 19,
  Formula: 20,
  DuplexLink: 21,
  Location: 22,
  GroupChat: 23,
  CreatedTime: 1001,
  ModifiedTime: 1002,
  CreatedUser: 1003,
  ModifiedUser: 1004,
  AutoNumber: 1005
}

// 保存cookie到本地存储
const saveCookieToStorage = (cookie) => {
  localStorage.setItem('xhs_cookie', cookie)
}

const loading = ref({
  fetchNotes: false,
  updateRecords: false
})
const showProgressDialog = ref(false)


const progress = ref({
  percent: 0,
  status: 'success',
  current: 0,
  total: 0,
  success: 0,
  failed: 0,
  skipped: 0,
  message: '准备开始处理...'
})

// 1. 定义 sleep 函数（支持随机时间）
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// 2. 生成 3～5 秒的随机等待时间（单位：毫秒）
function getRandomWaitTime() {
  const min = 3000; // 3秒
  const max = 5000; // 5秒
  return Math.floor(Math.random() * (max - min + 1)) + min;
}

// 检查API响应是否cookie过期
const checkCookieExpired = (response) => {
if (response.code !== 0 || ['登录', 'cookie', '过期'].some(str => response.msg.includes(str))) {
    saveCookieToStorage('')
    formData.value.cookie = ''
    return true
  }
  return false
}

const loadFromStorage = () => {
  return {
    cookie: localStorage.getItem('xhs_cookie') || '',
    author_url: localStorage.getItem('xhs_author_url') || '',
    last_cursor: localStorage.getItem('xhs_last_cursor') || ''
  }
}

// 保存数据到本地存储
const saveToStorage = (key, value) => {
  localStorage.setItem(`xhs_${key}`, value)
}

const formData = ref({
  ...loadFromStorage(), // 初始化时从存储加载
  likes_count: 0,
  max_count: 30
})

// 新增cursor输入框绑定
const cursorInput = ref('')

// 修改后的API调用函数
const callXhsAuthorNotesApi = async (cursor = '') => {
  try {
    const cleanCursor = typeof cursor === 'string' ? cursor : ''
    
    const response = await fetch('https://nibelungen.site/xhs/xhs_author_notes', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        url: formData.value.author_url,
        cookie: formData.value.cookie,
        cursor: cleanCursor
      }),
    })
    
    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`)

    const result = await response.json()
    
    if (checkCookieExpired(result)) {
      ElMessage.error('Cookie已过期，请更新Cookie')
      throw new Error('Cookie已过期')
    }
    
    if (result.code !== 0) throw new Error(result.msg || 'API调用失败')
    return result.data
  } catch (error) {
    console.error('API调用失败:', error)
    return null
  }
}

// 获取作者笔记函数
const fetchAuthorNotes = async (cursor = '', isContinue = false) => {
  const cleanCursor = isContinue ? cursorInput.value : cursor
  
  if (!formData.value.cookie) {
    updateProgress('请先输入小红书Cookie', 'exception')
    return
  }
  
  if (!formData.value.author_url && !isContinue) {
    updateProgress('请先输入作者主页URL', 'exception')
    return
  }

  try {
    loading.value.fetchNotes = true
    resetProgress()
    
    updateProgress('正在获取作者笔记...')
    
    const { tableId } = await bitable.base.getSelection()
    if (!tableId) {
      updateProgress('请先选择数据表', 'exception')
      loading.value.fetchNotes = false
      return
    }

    const table = await bitable.base.getTable(tableId)
    const allFields = await table.getFieldMetaList()
    
    // 确保关键字段存在
    const ensureField = async (fieldName, fieldType) => {
      let field = allFields.find(f => f.name === fieldName)
      if (!field) {
        updateProgress(`正在创建字段: ${fieldName}...`)
        field = await table.addField({
          type: fieldType,
          name: fieldName
        })
      }
      return field
    }

    const linkField = await ensureField('链接', FieldType.Url)
    const titleField = await ensureField('标题', FieldType.Text)
    const authorField = await ensureField('博主', FieldType.Text)
    const likesField = await ensureField('喜欢数', FieldType.Number)
    const coverField = await ensureField('封面图', FieldType.Url)
    const noteIdField = await ensureField('笔记ID', FieldType.Text)
    
    showProgressDialog.value = true
    
    let totalFetched = 0
    let currentCursor = cleanCursor
    let hasMore = true
    
    while (hasMore && totalFetched < formData.value.max_count) {
      const result = await callXhsAuthorNotesApi(currentCursor)
      if (!result || !result.items) {
        updateProgress('获取作者笔记失败', 'exception')
        break
      }
      
      // 保存当前cursor到本地存储
      saveToStorage('last_cursor', result.cursor || '')
      cursorInput.value = result.cursor || ''
      
      // 处理每个笔记
      const recordsToAdd = []
      for (const note of result.items) {
        if (totalFetched >= formData.value.max_count) break
        
        try {
          totalFetched++
          const likeCount = parseNumberWithUnits(note.like_count || "0")
          if (likeCount < formData.value.likes_count) {
            progress.value.skipped++
            updateProgress(`跳过笔记: 点赞数 ${likeCount} 小于设定值 ${formData.value.likes_count}`)
            continue
          }
          
          const recordData = {
            fields: {
              [linkField.id]: [{
                text: note.note_url,
                link: note.note_url,
                type: 'url'
              }],
              [titleField.id]: note.title || '',
              [authorField.id]: note.author_nickname || '',
              [likesField.id]: likeCount,
              [coverField.id]: [{
                text: '封面图',
                link: note.cover_url_default || note.cover_url_pre || '',
                type: 'url'
              }],
              [noteIdField.id]: note.note_id || ''
            }
          }
          
          recordsToAdd.push(recordData)          
        } catch (err) {
          progress.value.failed++
          console.error(`处理笔记失败:`, err)
          updateProgress(`处理笔记失败: ${err.message}`, 'exception')
        }
      }
      
      currentCursor = result.cursor || ''
      hasMore = result.has_more || false

      // 批量导入笔记
      if (recordsToAdd.length > 0) {
        updateProgress(`正在导入 ${recordsToAdd.length} 条笔记...`)
        
        try {
          const batchSize = 30
          for (let i = 0; i < recordsToAdd.length; i += batchSize) {
            const batch = recordsToAdd.slice(i, i + batchSize)
            await table.addRecords(batch)
            progress.value.success += batch.length
            updateProgress(`成功导入 ${batch.length} 条笔记 (${i + batch.length}/${recordsToAdd.length})`, 'success')
          }
        } catch (err) {
          console.error('批量导入失败:', err)
          progress.value.failed += recordsToAdd.length
          updateProgress(`部分笔记导入失败: ${err.message}`, 'warning')
        }
      }
      
      progress.value.current = totalFetched
      progress.value.total = formData.value.max_count
      updateProgressPercent()
      
      if (totalFetched >= formData.value.max_count) {
        updateProgress(`已达到最大获取数量 ${formData.value.max_count}`, 'success')
        break
      }
      
      if (!hasMore) {
        progress.value.total = totalFetched
        progress.value.current = progress.value.total
        updateProgressPercent()
        updateProgress('已获取全部笔记', 'success')
        break
      }

      const waitTime = getRandomWaitTime();
      await sleep(waitTime); 
    }
    
  } catch (error) {
    console.error('获取作者笔记失败:', error)
    updateProgress(`获取作者笔记失败: ${error.message}`, 'exception')
  } finally {
    loading.value.fetchNotes = false
  }
}

// 保存作者URL
const handleAuthorUrlChange = (value) => {
  formData.value.author_url = value
  saveToStorage('author_url', value)
  saveToStorage('last_cursor', '')
  cursorInput.value = ''
}

// 保存cookie
const handleCookieChange = (value) => {
  formData.value.cookie = value
  saveCookieToStorage(value)
}

// 调用小红书详情API
const callXhsDetailApi = async (url) => {
  try {
    const response = await fetch('https://nibelungen.site/xhs/xhs_detail', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        url,
        cookie: formData.value.cookie,
      }),
    })
    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`)

    const result = await response.json()
    
    if (checkCookieExpired(result)) {
      ElMessage.error('Cookie已过期，请更新Cookie')
      throw new Error('Cookie已过期')
    }
    
    if (result.code !== 0) throw new Error(result.msg || 'API调用失败')
    return result.data
  } catch (error) {
    console.error('API调用失败:', error)
    return null
  }
}

// 更新记录
const updateRecords = async () => {
  if (!formData.value.cookie) {
    updateProgress('请先输入小红书Cookie', 'exception')
    return
  }

  try {
    loading.value.updateRecords = true
    resetProgress()
    updateProgress('正在初始化处理...')

    const { tableId, viewId } = await bitable.base.getSelection()
    if (!tableId) {
      updateProgress('请先选择数据表', 'exception')
      loading.value.updateRecords = false
      return
    }

    const recordIds = await bitable.ui.selectRecordIdList(tableId, viewId)
    if (!recordIds?.length) {
      updateProgress('请选择要更新的记录', 'exception')
      loading.value.updateRecords = false
      return
    }

    const table = await bitable.base.getTable(tableId)
    const allFields = await table.getFieldMetaList()

    const linkFieldMeta = allFields.find(f => f.name === '链接')
    if (!linkFieldMeta) {
      updateProgress('未找到"链接"字段', 'exception')
      loading.value.updateRecords = false
      return
    }

    const linkField = await table.getFieldById(linkFieldMeta.id)
    const fieldValues = await linkField.getFieldValueList()
    const selectedRecords = fieldValues.filter(fv => recordIds.includes(fv.record_id))

    progress.value.total = selectedRecords.length
    showProgressDialog.value = true
    updateProgress(`开始处理 ${selectedRecords.length} 条记录...`)

    for (const fv of selectedRecords) {
      try {
        const recordId = fv.record_id
        const url = fv.value?.[0]?.text

        if (!url) {
          progress.value.skipped++
          updateProgress(`记录 ${recordId} 跳过：链接为空`)
          progress.value.current++
          updateProgressPercent()
          continue
        }

        updateProgress(`处理记录 ${recordId}: 获取数据中...`)
        const xhsData = await callXhsDetailApi(url)
        
        if (!xhsData) {
          progress.value.failed++
          updateProgress(`记录 ${recordId} 失败：接口返回为空`, 'warning')
        } else {
          updateProgress(`处理记录 ${recordId}: 格式化数据中...`)
          const updateFields = await formatXhsDataToFields(xhsData, allFields, table)

          if (Object.keys(updateFields).length > 0) {
            updateProgress(`处理记录 ${recordId}: 写入数据中...`)
            await table.setRecord(recordId, { fields: updateFields })
            progress.value.success++
            updateProgress(`记录 ${recordId} 更新成功`)
          } else {
            progress.value.skipped++
            updateProgress(`记录 ${recordId} 跳过：无匹配字段`)
          }
        }
      } catch (err) {
        progress.value.failed++
        console.error(`记录处理失败:`, err)
        updateProgress(`处理失败：${err.message}`, 'exception')
      } finally {
        progress.value.current++
        updateProgressPercent()
      }
    }

    updateProgress('处理完成！')
    if (progress.value.failed === 0 && progress.value.success > 0) {
      progress.value.status = 'success'
    } else if (progress.value.failed > 0 && progress.value.success > 0) {
      progress.value.status = 'warning'
    } else {
      progress.value.status = 'exception'
    }
  } catch (err) {
    console.error('全局错误：', err)
    updateProgress(`处理过程中发生错误: ${err.message}`, 'exception')
  } finally {
    loading.value.updateRecords = false
  }
}

// 格式化小红书数据为飞书字段
const formatXhsDataToFields = async (xhsData, allFields, table) => {
  const fieldMap = {}
  
  for (const field of allFields) {
    const fieldName = field.name.trim()
    
    switch (fieldName) {
      case '博主':
        try {
          const fieldInstance = await table.getField(field.id)
          const fieldType = await fieldInstance.getType()
          
          if (fieldType !== 3) {
            fieldMap[field.id] = xhsData.author
            break
          }

          const selectField = await table.getField(field.id)
          let options = await selectField.getOptions()
          
          const existingOption = options.find(opt => opt.name === xhsData.author)
          
          if (!existingOption) {
            await selectField.addOptions([{ name: xhsData.author }])
            options = await selectField.getOptions()
          }
          
          const selectedOption = options.find(opt => opt.name === xhsData.author)
          if (selectedOption) {
            fieldMap[field.id] = { id: selectedOption.id, text: selectedOption.name }
          }
        } catch (err) {
          console.error(`处理[博主]字段出错:`, err)
          fieldMap[field.id] = xhsData.author
        }
        break
        
      case '标题':
        fieldMap[field.id] = xhsData.title
        break
        
      case '文案':
        fieldMap[field.id] = xhsData.content
        break
        
      case '喜欢数':
        fieldMap[field.id] = parseNumberWithUnits(xhsData.like_count)
        break
        
      case '评论数':
        fieldMap[field.id] = parseNumberWithUnits(xhsData.comment_count)
        break
        
      case '收藏数':
        fieldMap[field.id] = parseNumberWithUnits(xhsData.collected_count)
        break
        
      case '分享数':
        fieldMap[field.id] = parseNumberWithUnits(xhsData.share_count)
        break
        
      case '发布地点':
        fieldMap[field.id] = xhsData.location
        break
        
      case '笔记发布时间':
        try {
          const timestamp = xhsData.publish_time || xhsData.lastUpdateTime
          const date = new Date(timestamp)
          
          const fieldInstance = await table.getField(field.id)
          const fieldType = await fieldInstance.getType()
          
          if (fieldType === FieldType.DateTime) {
            fieldMap[field.id] = date.getTime()
          } else {
            fieldMap[field.id] = xhsData.publish_time_format || 
              `${date.getFullYear()}-${(date.getMonth()+1).toString().padStart(2,'0')}-${date.getDate().toString().padStart(2,'0')} ` +
              `${date.getHours().toString().padStart(2,'0')}:${date.getMinutes().toString().padStart(2,'0')}:${date.getSeconds().toString().padStart(2,'0')}`
          }
        } catch (e) {
          console.error('处理发布时间出错:', e)
          fieldMap[field.id] = xhsData.publish_time_format || ''
        }
        break
        
      case '笔记标签词':
        try {
          const fieldInstance = await table.getField(field.id)
          const fieldType = await fieldInstance.getType()
          
          if (fieldType !== FieldType.MultiSelect) {
            fieldMap[field.id] = Array.isArray(xhsData.tag_list) 
              ? xhsData.tag_list.join(', ')
              : xhsData.tag_list || ''
            break
          }

          const multiSelectField = await table.getField(field.id)
          let options = await multiSelectField.getOptions()
          
          const rawTags = xhsData.tag_list || []
          const tags = Array.isArray(rawTags) ? rawTags : rawTags.split(/[,，]/)
          
          const newTags = tags.filter(tag => !options.some(opt => opt.name === tag))
          if (newTags.length > 0) {
            await multiSelectField.addOptions(newTags.map(name => ({ name })))
            options = await multiSelectField.getOptions()
          }
          
          fieldMap[field.id] = tags
            .map(tag => {
              const option = options.find(opt => opt.name === tag)
              return option ? { id: option.id, text: option.name } : null
            })
            .filter(Boolean)
            
        } catch (err) {
          console.error(`处理[笔记标签词]字段出错:`, err)
          fieldMap[field.id] = []
        }
        break
        
      case '笔记图片':
        if (xhsData.images_link?.length > 0) {
            fieldMap[field.id] = xhsData.images_link.map((url, index) => ({
                text: `图${index + 1}`,
                link: url,
                type: "url"
            }))
        } else {
            fieldMap[field.id] = ''
        }
      break
        
      case '视频保存':
        fieldMap[field.id] = xhsData.video_url ?? ''
        break
        
      default:
        break
    }
  }
  
  return fieldMap
}

// 重置进度
const resetProgress = () => {
  progress.value = {
    percent: 0,
    status: 'success',
    current: 0,
    total: 0,
    success: 0,
    failed: 0,
    skipped: 0,
    message: '准备开始处理...'
  }
}

// 更新进度信息
const updateProgress = (message, status = null) => {
  progress.value.message = message
  if (status) progress.value.status = status
}

// 更新进度百分比
const updateProgressPercent = () => {
  progress.value.percent = Math.min(
    100,
    Math.max(
      0,
      progress.value.total > 0
        ? Math.round((progress.value.current / progress.value.total) * 100)
        : 0
    )
  )
}

// 带单位的数字解析
function parseNumberWithUnits(input) {
  if (input == null || input === '') return 0
  if (typeof input === 'number') return input
  
  const str = String(input).trim()
  if (/^[\d,]+$/.test(str)) {
    return parseInt(str.replace(/,/g, ''), 10)
  }
  
  const unitMap = {
    '十': 10, '百': 100,
    '千': 1000, 'k': 1000, 'K': 1000,
    '万': 10000, 'w': 10000, 'W': 10000,
    '亿': 100000000,
    'm': 1000000, 'M': 1000000,
    'b': 1000000000, 'B': 1000000000
  }
  
  const match = str.match(/^([\d.,]+)\s*([^\d.]*)$/)
  if (!match) return 0
  
  const numPart = match[1].replace(/,/g, '')
  const unitPart = match[2].toLowerCase()
  const num = parseFloat(numPart)
  if (isNaN(num)) return 0
  
  if (!unitPart) return num
  if (unitPart.includes('万') || unitPart.includes('w')) return num * 10000
  
  for (const unit in unitMap) {
    if (unitPart.includes(unit.toLowerCase())) {
      return num * unitMap[unit]
    }
  }
  
  return num
}

onMounted(() => {
  document.title = '小红书助手 - 详情批量更新'
  
  if (formData.value.cookie) {
    ElMessage.success('已加载保存的Cookie')
  }
})
</script>

<template>
  <div class="container">
    <el-alert title="使用说明" type="info" :closable="false">
      1. 输入小红书Cookie和作者主页URL<br />
      2. 设置筛选条件（点赞数、获取数量）<br />
      3. 获取笔记后会自动导入到表格<br />
      4. 中断后可输入cursor继续获取<br />
      5. 也可以直接更新已有链接的笔记数据
    </el-alert>

    <el-divider />

    <el-card class="config-card">
      <el-form label-position="top">
        <el-form-item label="小红书Cookie" required>
          <el-input
            v-model="formData.cookie"
            @change="handleCookieChange"
            placeholder="输入小红书Cookie"
            show-password
            clearable
          >
            <template #append>
              <el-button :icon="Refresh" @click="formData.cookie = ''" />
            </template>
          </el-input>
        </el-form-item>
        
        <el-form-item label="作者主页URL" required>
          <el-input
            v-model="formData.author_url"
            @change="handleAuthorUrlChange"
            placeholder="输入小红书作者主页URL"
            clearable
          />
        </el-form-item>
        
        <el-form-item label="Cursor (用于继续翻页获取)">
          <el-input
            v-model="cursorInput"
            placeholder="输入上次获取的cursor"
            clearable
          />
        </el-form-item>
        
        <div class="amount-area">
          <el-form-item label="最小点赞数">
            <el-input-number
              v-model="formData.likes_count"
              :min="0"
              :max="100000"
              :step="100"
            />
          </el-form-item>
          
          <el-form-item label="最大获取数量">
            <el-input-number
              v-model="formData.max_count"
              :min="1"
              :max="2000"
              :step="30"
            />
          </el-form-item>
        </div>
      </el-form>

      <div class="action-area">
        <el-button
          type="primary"
          @click="() => fetchAuthorNotes(cursorInput, false)"
          :loading="loading.fetchNotes"
          :disabled="!formData.cookie || !formData.author_url"
          size="large"
        >
          <el-icon><CircleCheck /></el-icon>
          获取作者笔记
        </el-button>
        
        <el-button
          type="success"
          @click="updateRecords"
          :loading="loading.updateRecords"
          :disabled="!formData.cookie"
          size="large"
        >
          <el-icon><CircleCheck /></el-icon>
          批量更新笔记详情
        </el-button>
      </div>
    </el-card>

    <el-dialog
      v-model="showProgressDialog"
      title="处理进度"
      width="400px"
      :close-on-click-modal="false"
      :close-on-press-escape="false"
      :show-close="false"
      custom-class="centered-dialog"  
    >
      <div class="dialog-content">
        <water-tank-progress
          :percent="progress.percent"
          :status="progress.status"
          :size="180"
          class="progress-circle"
        />
        <div class="progress-details">
          <div class="progress-message">{{ progress.message }}</div>
          <div class="progress-stats">
            <div class="stats-row">
              <div>进度: {{ progress.current }}/{{ progress.total }}</div>
              <span class="success">成功: {{ progress.success }}</span>
              <span class="failed">失败: {{ progress.failed }}</span>
              <span class="skipped">跳过: {{ progress.skipped }}</span>
            </div>
          </div>
        </div>
      </div>

      <template #footer>
        <div class="dialog-footer">
          <el-button
            type="primary"
            @click="showProgressDialog = false"
            :disabled="loading.fetchNotes || loading.updateRecords"
          >
            关闭
          </el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<style scoped>
.centered-dialog .el-dialog__body {
  display: flex;
  flex-direction: column;
  align-items: center;  /* 水平居中 */
  padding: 20px 25px;   /* 调整内边距 */
}

/* 内容容器 */
.dialog-content {
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;  /* 元素间距 */
}

/* 进度圆圈 */
.progress-circle {
  margin: 0 auto;  /* 双重保障居中 */
}

/* 进度详情 */
.progress-details {
  text-align: center;
  width: 100%;
}

/* 统计行 */
.stats-row {
  display: flex;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 8px 16px;
  margin-top: 8px;
}

/* 底部按钮居中 */
.dialog-footer {
  text-align: center;
}

.config-card {
  margin-bottom: 20px;
}

.amount-area {
    display: flex;
    justify-content: flex-start;
    margin-top: 20px;
    gap: 20px;
}

.action-area {
  display: flex;
  justify-content: space-between;
  margin-top: 20px;
  gap: 20px;
}

.success {
  color: #67c23a;
  font-weight: bold;
}

.failed {
  color: #f56c6c;
  font-weight: bold;
}

.skipped {
  color: #909399;
  font-weight: bold;
}

.note-link {
  font-family: monospace;
  word-break: break-all;
  margin-right: 10px;
  display: inline-block;
  max-width: calc(100% - 80px);
  overflow: hidden;
  text-overflow: ellipsis;
  vertical-align: middle;
}

.copy-btn {
  margin-left: 10px;
}

.el-table {
  margin-top: 10px;
}

.el-dialog__body {
  padding-top: 10px;
}

.pagination-wrapper {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 15px;
}
</style>