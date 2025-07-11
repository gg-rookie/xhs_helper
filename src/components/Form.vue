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

// 从本地存储加载cookie
const loadCookieFromStorage = () => {
  return localStorage.getItem('xhs_cookie') || ''
}

// 保存cookie到本地存储
const saveCookieToStorage = (cookie) => {
  localStorage.setItem('xhs_cookie', cookie)
}

const formData = ref({
  cookie: loadCookieFromStorage(),
  author_url: '',
  likes_count: 0,
  max_count: 30
})

const loading = ref(false)
const showProgressDialog = ref(false)
const showAuthorNotesDialog = ref(false)
const authorNotes = ref([])
const pagination = ref({
  cursor: '',
  has_more: false,
  current_page: 1,
  total: 0
})

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

// 检查API响应是否cookie过期
const checkCookieExpired = (response) => {
  if (response.code === -1 && response.msg.includes('登录')) {
    return true
  }
  return false
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

// 调用小红书作者笔记API
const callXhsAuthorNotesApi = async (cursor = '') => {
  try {
    const response = await fetch('https://nibelungen.site/xhs/xhs_author_notes_1000', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        url: formData.value.author_url,
        cookie: formData.value.cookie,
        likes_count: formData.value.likes_count,
        max_count: formData.value.max_count,
        cursor: cursor
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

// 获取作者笔记
const fetchAuthorNotes = async (cursor = '', isLoadMore = false) => {
  if (!formData.value.cookie) {
    updateProgress('请先输入小红书Cookie', 'exception')
    return
  }
  
  if (!formData.value.author_url) {
    updateProgress('请先输入作者主页URL', 'exception')
    return
  }

  try {
    loading.value = true
    if (!isLoadMore) {
      resetProgress()
      authorNotes.value = []
      pagination.value.current_page = 1
    }
    
    updateProgress('正在获取作者笔记...')
    
    const result = await callXhsAuthorNotesApi(cursor)
    if (!result || !result.notes) {
      updateProgress('获取作者笔记失败', 'exception')
      return
    }
    
    const newNotes = result.notes.map(noteUrl => ({
      rawUrl: noteUrl,
      displayUrl: noteUrl
    }))
    
    if (isLoadMore) {
      authorNotes.value = [...authorNotes.value, ...newNotes]
    } else {
      authorNotes.value = newNotes
    }
    
    // 更新分页信息
    pagination.value = {
      cursor: result.cursor || '',
      has_more: result.has_more || false,
      current_page: isLoadMore ? pagination.value.current_page + 1 : 1,
      total: isLoadMore ? pagination.value.total + newNotes.length : newNotes.length
    }
    
    showAuthorNotesDialog.value = true
    updateProgress('获取作者笔记成功', 'success')
    
  } catch (error) {
    console.error('获取作者笔记失败:', error)
    updateProgress(`获取作者笔记失败: ${error.message}`, 'exception')
  } finally {
    loading.value = false
  }
}

// 加载更多笔记
const loadMoreNotes = () => {
  if (pagination.value.has_more && pagination.value.cursor) {
    fetchAuthorNotes(pagination.value.cursor, true)
  }
}

// 保存cookie
const handleCookieChange = (value) => {
  formData.value.cookie = value
  saveCookieToStorage(value)
}

// 更新记录
const updateRecords = async () => {
  if (!formData.value.cookie) {
    updateProgress('请先输入小红书Cookie', 'exception')
    return
  }

  try {
    loading.value = true
    resetProgress()
    updateProgress('正在初始化处理...')

    const { tableId, viewId } = await bitable.base.getSelection()
    if (!tableId) {
      updateProgress('请先选择数据表', 'exception')
      loading.value = false
      return
    }

    const recordIds = await bitable.ui.selectRecordIdList(tableId, viewId)
    if (!recordIds?.length) {
      updateProgress('请选择要更新的记录', 'exception')
      loading.value = false
      return
    }

    const table = await bitable.base.getTable(tableId)
    const allFields = await table.getFieldMetaList()

    const linkFieldMeta = allFields.find(f => f.name === '链接')
    if (!linkFieldMeta) {
      updateProgress('未找到"链接"字段', 'exception')
      loading.value = false
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
    loading.value = false
  }
}

// 格式化小红书数据为飞书字段
const formatXhsDataToFields = async (xhsData, allFields, table) => {
  const fieldMap = {}
  
  for (const field of allFields) {
    const fieldName = field.name.trim()
    
    switch (fieldName) {
      case '博主':  // 处理单选字段
        try {
          const fieldInstance = await table.getField(field.id)
          const fieldType = await fieldInstance.getType()
          
          if (fieldType !== 3) {  // 如果不是单选字段，直接赋值文本
            fieldMap[field.id] = xhsData.author
            break
          }

          // 获取单选字段的选项
          const selectField = await table.getField(field.id)
          let options = await selectField.getOptions()
          
          // 检查选项是否已存在
          const existingOption = options.find(opt => opt.name === xhsData.author)
          
          if (!existingOption) {
            // 添加新选项
            await selectField.addOptions([{ name: xhsData.author }])
            options = await selectField.getOptions()  // 重新获取选项
          }
          
          // 设置字段值
          const selectedOption = options.find(opt => opt.name === xhsData.author)
          if (selectedOption) {
            fieldMap[field.id] = { id: selectedOption.id, text: selectedOption.name }
          }
        } catch (err) {
          console.error(`处理[博主]字段出错:`, err)
          fieldMap[field.id] = xhsData.author  // 出错时回退到文本
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

// 导入作者笔记到表格
const importAuthorNotesToTable = async () => {
  if (!authorNotes.value.length) {
    updateProgress('没有可导入的笔记数据', 'warning')
    return
  }

  try {
    loading.value = true
    resetProgress()
    updateProgress('准备导入笔记数据...')
    
    const { tableId, viewId } = await bitable.base.getSelection()
    if (!tableId) {
      updateProgress('请先选择数据表', 'exception')
      loading.value = false
      return
    }

    const table = await bitable.base.getTable(tableId)
    const allFields = await table.getFieldMetaList()

    let linkField = allFields.find(f => f.name === '链接')
    if (!linkField) {
      updateProgress('正在创建链接字段...')
      linkField = await table.addField({
        type: FieldType.Url,
        name: '链接'
      })
    }

    progress.value.total = authorNotes.value.length
    showProgressDialog.value = true
    updateProgress(`开始导入 ${authorNotes.value.length} 条笔记...`)

    const batchSize = 10
    for (let i = 0; i < authorNotes.value.length; i += batchSize) {
      const batchNotes = authorNotes.value.slice(i, i + batchSize)
      
      const records = batchNotes.map(note => ({
        fields: {
          [linkField.id]: [{
            text: note.displayUrl,
            link: note.rawUrl,
            type: 'url'
          }]
        }
      }))

      try {
        updateProgress(`正在导入第 ${i + 1}-${i + batchNotes.length} 条笔记...`)
        await table.addRecords(records)
        progress.value.success += batchNotes.length
      } catch (err) {
        console.error('批量导入失败:', err)
        progress.value.failed += batchNotes.length
        updateProgress(`部分笔记导入失败: ${err.message}`, 'warning')
        
        for (const note of batchNotes) {
          try {
            await table.addRecord({
              fields: {
                [linkField.id]: [{
                  text: note.displayUrl,
                  link: note.rawUrl
                }]
              }
            })
            progress.value.success++
          } catch (e) {
            progress.value.failed++
            console.error('单条导入失败:', note.rawUrl, e)
          }
          progress.value.current++
          updateProgressPercent()
        }
      } finally {
        progress.value.current += batchNotes.length
        updateProgressPercent()
      }
    }

    updateProgress('笔记导入完成!', progress.value.failed > 0 ? 'warning' : 'success')
  } catch (err) {
    console.error('导入笔记出错:', err)
    updateProgress(`导入失败: ${err.message}`, 'exception')
  } finally {
    loading.value = false
    showAuthorNotesDialog.value = false
  }
}

// 复制到剪贴板
const copyToClipboard = (text) => {
  navigator.clipboard.writeText(text)
    .then(() => {
      ElMessage.success('已复制到剪贴板')
    })
    .catch(err => {
      console.error('复制失败:', err)
      ElMessage.error('复制失败')
    })
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
  
  // 检查cookie是否存在
  if (formData.value.cookie) {
    ElMessage.success('已加载保存的Cookie')
  }
})
</script>

<template>
  <div class="container">
    <h1>小红书助手 - 详情批量更新</h1>

    <el-alert title="使用说明" type="info" :closable="false">
      1. 输入小红书Cookie和作者主页URL<br />
      2. 设置筛选条件（点赞数、获取数量）<br />
      3. 获取笔记后可以导入到表格<br />
      4. 也可以直接更新已有链接的笔记数据
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
            placeholder="输入小红书作者主页URL"
            clearable
          />
        </el-form-item>
        
        <el-form-item label="最小点赞数">
          <el-input-number
            v-model="formData.likes_count"
            :min="0"
            :max="10000"
            :step="100"
          />
        </el-form-item>
        
        <el-form-item label="最大获取数量">
          <el-input-number
            v-model="formData.max_count"
            :min="1"
            :max="1000"
            :step="10"
          />
        </el-form-item>
      </el-form>

      <div class="action-area">
        <el-button
          type="primary"
          @click="fetchAuthorNotes"
          :loading="loading"
          :disabled="!formData.cookie || !formData.author_url"
          size="large"
        >
          <el-icon><CircleCheck /></el-icon>
          获取作者笔记
        </el-button>
        
        <el-button
          type="success"
          @click="updateRecords"
          :loading="loading"
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
    >
      <water-tank-progress
        :percent="progress.percent"
        :status="progress.status"
        :size="180"
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

      <template #footer>
        <el-button
          type="primary"
          @click="showProgressDialog = false"
          :disabled="loading"
        >
          关闭
        </el-button>
      </template>
    </el-dialog>
    
    <el-dialog
      v-model="showAuthorNotesDialog"
      title="作者笔记列表"
      width="80%"
      top="5vh"
    >
      <el-table 
        :data="authorNotes" 
        border 
        style="width: 100%"
        height="60vh"
      >
        <el-table-column prop="displayUrl" label="笔记链接" min-width="500">
          <template #default="{ row }">
            <el-link 
              type="primary" 
              :href="row.rawUrl" 
              target="_blank" 
              :underline="false"
              class="note-link"
            >
              {{ row.displayUrl }}
            </el-link>
            <el-button
              size="small"
              @click="copyToClipboard(row.rawUrl)"
              class="copy-btn"
            >
              复制
            </el-button>
          </template>
        </el-table-column>
      </el-table>
      
      <div class="pagination-wrapper">
        <el-pagination
          small
          layout="prev, pager, next"
          :page-size="20"
          :total="pagination.total"
          :current-page="pagination.current_page"
          @current-change="fetchAuthorNotes"
          hide-on-single-page
        />
        
        <el-button
          v-if="pagination.has_more"
          type="primary"
          @click="loadMoreNotes"
          :loading="loading"
          class="load-more-btn"
        >
          加载更多
        </el-button>
      </div>
      
      <template #footer>
        <el-button
          type="primary"
          @click="importAuthorNotesToTable"
          :loading="loading"
        >
          导入到表格
        </el-button>
        <el-button
          @click="showAuthorNotesDialog = false"
        >
          关闭
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<style scoped>
.container {
  padding: 20px;
  max-width: 1000px;
  margin: 0 auto;
}

.config-card {
  margin-bottom: 20px;
}

.action-area {
  display: flex;
  justify-content: flex-start;
  margin-top: 20px;
  gap: 20px;
}

.progress-details {
  margin-top: 20px;
  text-align: center;
}

.progress-message {
  font-size: 16px;
  margin-bottom: 10px;
  font-weight: bold;
}

.progress-stats {
  font-size: 14px;
}

.stats-row {
  display: flex;
  justify-content: space-between;
  margin-top: 10px;
  flex-wrap: wrap;
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

.load-more-btn {
  margin-left: 15px;
}
</style>