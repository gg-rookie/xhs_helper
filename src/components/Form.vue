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
  ElInputNumber
} from 'element-plus'
import { CircleCheck, Close } from '@element-plus/icons-vue'

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
  message: '准备开始处理...'
})

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
    
    authorNotes.value = result.notes.map(noteUrl => ({
      rawUrl: noteUrl,
      displayUrl: noteUrl.split('?')[0] // 去除参数显示
    }))
    
    showAuthorNotesDialog.value = true
    updateProgress('获取作者笔记成功', 'success')
    
  } catch (error) {
    console.error('获取作者笔记失败:', error)
    updateProgress(`获取作者笔记失败: ${error.message}`, 'exception')
  } finally {
    loading.value = false
  }
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

    // 确保链接字段存在
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

    // 分批处理（每次处理10条）
    const batchSize = 10
    for (let i = 0; i < authorNotes.value.length; i += batchSize) {
      const batchNotes = authorNotes.value.slice(i, i + batchSize)
      
      const records = batchNotes.map(note => {
        return {
          fields: {
            [linkField.id]: [{
              text: note.displayUrl,  // 显示原始链接
              link: note.rawUrl,      // 保留完整链接
              type: 'url'
            }]
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
        
        // 失败后尝试单条导入
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
        height="70vh"
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
</style>