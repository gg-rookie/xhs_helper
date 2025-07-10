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
  ElSlider,
  ElInputNumber,
  ElTable,
  ElTableColumn,
  ElLink,
} from 'element-plus'
import { CircleCheck, Close } from '@element-plus/icons-vue'

const formData = ref({ 
  cookie: '',
  author_url: '',
  likes_count: 0,
  max_count: 30
})
const loading = ref(false)
const showProgressDialog = ref(false)
const showAuthorNotesDialog = ref(false)
const authorNotes = ref([])
const progress = ref({
  percent: 0,
  status: 'success',
  current: 0,
  total: 0,
  success: 0,
  failed: 0,
  skipped: 0,
  message: '准备开始处理...',
})

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
    if (result.code !== 0) throw new Error(result.msg || 'API调用失败')
    return result.data
  } catch (error) {
    console.error('API调用失败:', error)
    return null
  }
}

// 调用小红书作者笔记API
const callXhsAuthorNotesApi = async () => {
  try {
    const response = await fetch('https://nibelungen.site/xhs/xhs_author_notes_1000', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        url: formData.value.author_url,
        cookie: formData.value.cookie,
        likes_count: formData.value.likes_count,
        max_count: formData.value.max_count,
      }),
    })
    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`)
    const result = await response.json()
    if (result.code !== 0) throw new Error(result.msg || 'API调用失败')
    return result.data
  } catch (error) {
    console.error('API调用失败:', error)
    return null
  }
}

// 获取作者笔记
const fetchAuthorNotes = async () => {
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
    resetProgress()
    updateProgress('正在获取作者笔记...')
    
    const result = await callXhsAuthorNotesApi()
    if (!result || !result.notes) {
      updateProgress('获取作者笔记失败', 'exception')
      return
    }
    
    authorNotes.value = result.notes
    showAuthorNotesDialog.value = true
    updateProgress('获取作者笔记成功', 'success')
    
  } catch (error) {
    console.error('获取作者笔记失败:', error)
    updateProgress(`获取作者笔记失败: ${error.message}`, 'exception')
  } finally {
    loading.value = false
  }
}

// 将作者笔记导入到表格
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

    // 查找链接字段
    const linkField = allFields.find(f => f.name === '链接')
    if (!linkField) {
      updateProgress('未找到"链接"字段，请确保表格中有链接字段', 'exception')
      loading.value = false
      return
    }

    progress.value.total = authorNotes.value.length
    showProgressDialog.value = true
    updateProgress(`开始导入 ${authorNotes.value.length} 条笔记...`)

    // 批量添加记录
    const batchSize = 10  // 每次批量处理10条
    for (let i = 0; i < authorNotes.value.length; i += batchSize) {
      const batchNotes = authorNotes.value.slice(i, i + batchSize)
      
      // 准备批量添加的数据
      const records = batchNotes.map(note => {
        return {
          fields: {
            [linkField.id]: [{ text: note.url, type: 'url' }],
            // 可以添加其他字段的初始值
          }
        }
      })

      try {
        updateProgress(`正在导入第 ${i + 1}-${i + batchNotes.length} 条笔记...`)
        await table.addRecords(records)
        progress.value.success += batchNotes.length
      } catch (err) {
        console.error('批量导入失败:', err)
        progress.value.failed += batchNotes.length
        updateProgress(`部分笔记导入失败: ${err.message}`, 'warning')
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
    message: '准备开始处理...',
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

// 强制使用JPG格式
function getMimeTypeFromUrl() {
  return 'image/jpeg'
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

// 上传文件到飞书并获取token（强制JPG格式）
const uploadFileToBitable = async (url) => {
  try {
    const proxyUrl = `https://nibelungen.site/xhs/proxy-image?url=${encodeURIComponent(url)}`
    const response = await fetch(proxyUrl)
    if (!response.ok) throw new Error(`Failed to fetch image: ${response.status}`)
    
    const blob = await response.blob()
    const filename = url.split('/').pop()?.replace(/\.[^/.]+$/, '') || `image_${Date.now()}`
    const jpgFilename = `${filename}.jpg`
    
    return {
      token: await convertBlobToJpgToken(blob, jpgFilename),
      name: jpgFilename,
      type: 'image/jpeg',
      timeStamp: Date.now(),
      size: blob.size
    }
  } catch (error) {
    console.error('文件上传失败:', error)
    throw error
  }
}

// 将Blob转换为JPG格式并上传
const convertBlobToJpgToken = async (blob, filename) => {
  try {
    const file = blob.type === 'image/jpeg' 
      ? new File([blob], filename, { type: 'image/jpeg' })
      : new File([blob], filename, { type: 'image/jpeg' })
    
    const tokens = await bitable.base.batchUploadFile([file])
    if (!tokens || !tokens.length) throw new Error('No file token returned')
    return tokens[0]
  } catch (error) {
    console.error('JPG转换上传失败:', error)
    throw error
  }
}

// 格式化小红书数据为飞书字段
const formatXhsDataToFields = async (xhsData, allFields, table) => {
  const fieldMap = {}
  
  for (const field of allFields) {
    const fieldName = field.name.trim()
    
    switch (fieldName) {
      case '博主':
        fieldMap[field.id] = xhsData.author
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
        // fieldMap[field.id] = new Date(xhsData.publish_time).toISOString()
        fieldMap[field.id] = new Date(xhsData.publish_time).getTime();
        break
        
      case '笔记标签词':
        try {
          const fieldInstance = await table.getField(field.id)
          const fieldType = await fieldInstance.getType()
          
          if (fieldType !== 4) {
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
          try {
            const attachments = []
            
            for (const url of xhsData.images_link.slice(0, 10)) {
              try {
                const attachment = await uploadFileToBitable(url)
                attachments.push(attachment)
                await new Promise(resolve => setTimeout(resolve, 500))
              } catch (e) {
                console.error(`处理图片失败: ${url}`, e)
              }
            }
            
            fieldMap[field.id] = attachments
          } catch (err) {
            console.error('处理附件字段失败:', err)
            fieldMap[field.id] = []
          }
        } else {
          fieldMap[field.id] = []
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

onMounted(() => {
  document.title = '小红书助手 - 详情批量更新'
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
            placeholder="输入小红书Cookie"
            show-password
            clearable
          />
        </el-form-item>
        
        <el-form-item label="作者主页URL" required>
          <el-input
            v-model="formData.author_url"
            placeholder="输入小红书作者主页URL"
            clearable
          />
        </el-form-item>
        
        <el-form-item label="最小点赞数">
          <el-slider
            v-model="formData.likes_count"
            :min="0"
            :max="10000"
            :step="100"
            show-input
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
          更新笔记数据
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
          icon="Close"
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
        height="70vh"
        highlight-current-row
      >
        <el-table-column prop="title" label="标题" width="200" />
        <el-table-column prop="like_count" label="点赞数" width="100" align="center" />
        <el-table-column prop="comment_count" label="评论数" width="100" align="center" />
        <el-table-column prop="collected_count" label="收藏数" width="100" align="center" />
        <el-table-column prop="publish_time" label="发布时间" width="180" />
        <el-table-column prop="url" label="链接">
          <template #default="{ row }">
            <el-link type="primary" :href="row.url" target="_blank" :underline="false">
              {{ row.url }}
            </el-link>
          </template>
        </el-table-column>
      </el-table>
      
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
          icon="Close"
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
  justify-content: space-between;
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

.el-table {
  margin-top: 10px;
}

.el-dialog__body {
  padding-top: 10px;
}
</style>