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
} from 'element-plus'
import { CircleCheck, Close } from '@element-plus/icons-vue'

const formData = ref({ cookie: '' })
const loading = ref(false)
const showProgressDialog = ref(false)
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

// 根据URL获取MIME类型
function getMimeTypeFromUrl(url) {
  const extension = url.split('.').pop()?.toLowerCase()
  switch(extension) {
    case 'jpg':
    case 'jpeg': return 'image/jpeg'
    case 'png': return 'image/png'
    case 'gif': return 'image/gif'
    case 'mp4': return 'video/mp4'
    default: return 'application/octet-stream'
  }
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

// 上传文件到飞书并获取token
const uploadFileToBitable = async (url) => {
  try {
    const proxyUrl = `https://nibelungen.site/xhs/proxy-image?url=${encodeURIComponent(url)}`
    const response = await fetch(proxyUrl)
    if (!response.ok) throw new Error(`Failed to fetch image: ${response.status}`)
    
    const blob = await response.blob()
    const filename = url.split('/').pop() || `image_${Date.now()}.${blob.type.split('/')[1] || 'jpg'}`
    const file = new File([blob], filename, { type: blob.type })
    
    // 上传文件并获取token
    const tokens = await bitable.base.batchUploadFile([file])
    if (!tokens || !tokens.length) throw new Error('No file token returned')
    
    return {
      token: tokens[0],
      name: filename,
      type: blob.type,
      timeStamp: Date.now(),
      size: blob.size
    }
  } catch (error) {
    console.error('文件上传失败:', error)
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
        fieldMap[field.id] = new Date(xhsData.publish_time).toISOString()
        break
        
      case '笔记标签词':
        try {
          const fieldInstance = await table.getField(field.id)
          const fieldType = await fieldInstance.getType()
          
          if (fieldType !== 4) { // 非多选字段
            fieldMap[field.id] = Array.isArray(xhsData.tag_list) 
              ? xhsData.tag_list.join(', ')
              : xhsData.tag_list || ''
            break
          }

          // 多选字段处理
          const multiSelectField = await table.getField(field.id)
          let options = await multiSelectField.getOptions()
          
          const rawTags = xhsData.tag_list || []
          const tags = Array.isArray(rawTags) ? rawTags : rawTags.split(/[,，]/)
          
          // 添加新标签选项
          const newTags = tags.filter(tag => !options.some(opt => opt.name === tag))
          if (newTags.length > 0) {
            await multiSelectField.addOptions(newTags.map(name => ({ name })))
            options = await multiSelectField.getOptions() // 刷新选项
          }
          
          // 构建字段值
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
            // 上传所有图片并获取token
            const attachments = []
            
            // 限制最多处理10张图片
            for (const url of xhsData.images_link.slice(0, 10)) {
              try {
                const attachment = await uploadFileToBitable(url)
                attachments.push(attachment)
                
                // 添加延迟避免触发速率限制
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

    // 获取当前选择的表格和视图
    const { tableId, viewId } = await bitable.base.getSelection()
    if (!tableId) {
      updateProgress('请先选择数据表', 'exception')
      loading.value = false
      return
    }

    // 选择要更新的记录
    const recordIds = await bitable.ui.selectRecordIdList(tableId, viewId)
    if (!recordIds?.length) {
      updateProgress('请选择要更新的记录', 'exception')
      loading.value = false
      return
    }

    // 获取表格和字段信息
    const table = await bitable.base.getTable(tableId)
    const allFields = await table.getFieldMetaList()

    // 检查链接字段是否存在
    const linkFieldMeta = allFields.find(f => f.name === '链接')
    if (!linkFieldMeta) {
      updateProgress('未找到"链接"字段', 'exception')
      loading.value = false
      return
    }

    // 获取选中记录的链接
    const linkField = await table.getFieldById(linkFieldMeta.id)
    const fieldValues = await linkField.getFieldValueList()
    const selectedRecords = fieldValues.filter(fv => recordIds.includes(fv.record_id))

    // 设置进度信息
    progress.value.total = selectedRecords.length
    showProgressDialog.value = true
    updateProgress(`开始处理 ${selectedRecords.length} 条记录...`)

    // 处理每条记录
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

        // 调用API获取小红书数据
        updateProgress(`处理记录 ${recordId}: 获取数据中...`)
        const xhsData = await callXhsDetailApi(url)
        
        if (!xhsData) {
          progress.value.failed++
          updateProgress(`记录 ${recordId} 失败：接口返回为空`, 'warning')
        } else {
          // 格式化数据为飞书字段
          updateProgress(`处理记录 ${recordId}: 格式化数据中...`)
          const updateFields = await formatXhsDataToFields(xhsData, allFields, table)

          // 更新记录
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

    // 处理完成
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
    <h1>小红书详情批量更新</h1>

    <el-alert title="使用说明" type="info" :closable="false">
      1. 输入小红书Cookie <br />
      2. 点击按钮后选择要更新的记录 <br />
      3. 系统将自动匹配并更新所有可用字段
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
      </el-form>

      <div class="action-area">
        <el-button
          type="primary"
          @click="updateRecords"
          :loading="loading"
          :disabled="!formData.cookie"
          size="large"
        >
          <el-icon><CircleCheck /></el-icon>
          开始更新数据
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
  </div>
</template>

<style scoped>
.container {
  padding: 20px;
  max-width: 800px;
  margin: 0 auto;
}

h1 {
  text-align: center;
  color: #ff2442;
  margin-bottom: 20px;
}

.config-card {
  margin-top: 20px;
}

.action-area {
  margin-top: 20px;
  text-align: center;
}

.progress-details {
  margin-top: 20px;
  padding: 15px 20px;
  background-color: #f8f8f8;
  border-radius: 4px;
  max-width: 100%;
  box-sizing: border-box;
  word-break: break-word;
}

.progress-message {
  font-size: 14px;
  margin-bottom: 10px;
  color: #333;
  white-space: normal;
}

.progress-stats {
  font-size: 14px;
  display: flex;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 10px;
}

.progress-stats > div:first-child {
  flex: 1 1 100%;
  margin-bottom: 8px;
}

.stats-row {
  display: flex;
  gap: 15px;
  flex-wrap: wrap;
  flex: 1 1 auto;
}

.stats-row span {
  white-space: nowrap;
}

.success {
  color: #67c23a;
}

.failed {
  color: #f56c6c;
}

.skipped {
  color: #e6a23c;
}

::v-deep(.el-dialog__body) {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 20px 0;
}

::v-deep(water-tank-progress) {
  margin: 0 auto 20px;
}

::v-deep(.progress-details) {
  width: 100%;
  max-width: 320px;
  text-align: center;
}
</style>