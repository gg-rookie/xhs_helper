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

// 这里根据你的表结构替换成实际fieldId
const fieldId = 'yourFieldId'

const formData = ref({ cookie: '' })
const loading = ref(false)
const showProgressDialog = ref(false)
const progress = ref({
  percent: 0,
  status: 'success', // success, warning, exception
  current: 0,
  total: 0,
  success: 0,
  failed: 0,
  skipped: 0,
  message: '准备开始处理...',
})

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

const updateProgress = (message, status = null) => {
  progress.value.message = message
  if (status) progress.value.status = status
}

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

const formatXhsDataToFields = (xhsData, allFields) => {
  const fieldMap = {}
  for (const field of allFields) {
    switch (field.name) {
      case '博主':
        fieldMap[field.id] = xhsData.author
        break
      case '标题':
        fieldMap[field.id] = xhsData.title
        break
      case '文案':
        fieldMap[field.id] = xhsData.content
        break
      case '点赞数':
        fieldMap[field.id] = xhsData.like_count
        break
      case '评论数':
        fieldMap[field.id] = xhsData.comment_count
        break
      case '收藏数':
        fieldMap[field.id] = xhsData.collected_count
        break
      case '分享数':
        fieldMap[field.id] = xhsData.share_count
        break
      case '发布地点':
        fieldMap[field.id] = xhsData.location
        break
      case '笔记发布时间':
        // 时间戳转 ISO 字符串或 Date 对象
        fieldMap[field.id] = new Date(xhsData.publish_time).toISOString()
        break
      case '笔记标签词':
        fieldMap[field.id] = xhsData.tag_list?.join(', ') ?? ''
        break
      case '笔记图片':
        fieldMap[field.id] = xhsData.images_link?.slice(0, 3).join(', ') ?? ''
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
      updateProgress('未找到“链接”字段', 'exception')
      loading.value = false
      return
    }

    const linkField = await table.getFieldById(linkFieldMeta.id)
    const fieldValues = await linkField.getFieldValueList()

    // 只保留选中的记录的链接
    const selectedRecords = fieldValues.filter(fv => recordIds.includes(fv.record_id))
    console.log(selectedRecords);

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

        const xhsData = await callXhsDetailApi(url)
        if (!xhsData) {
          progress.value.failed++
          updateProgress(`记录 ${recordId} 失败：接口返回为空`)
        } else {
          // const updateFields = {}
          // for (const field of allFields) {
          //   if (xhsData[field.name] !== undefined) {
          //     updateFields[field.id] = xhsData[field.name]
          //   }
          // }
          const updateFields = formatXhsDataToFields(xhsData, allFields)


          if (Object.keys(updateFields).length > 0) {
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
        console.log(err)
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
      width="320px"
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
  max-width: 100%;       /* 限制宽度不要超过弹窗 */
  box-sizing: border-box;
  word-break: break-word; /* 防止长文本溢出 */
}

.progress-message {
  font-size: 14px;
  margin-bottom: 10px;
  color: #333;
  white-space: normal;    /* 允许换行 */
}

.progress-stats {
  font-size: 14px;
  display: flex;
  justify-content: space-between;
  flex-wrap: wrap;        /* 小屏幕换行 */
  gap: 10px;              /* 各项间距 */
}

.progress-stats > div:first-child {
  flex: 1 1 100%;        /* 进度信息独占一行 */
  margin-bottom: 8px;
}

.stats-row {
  display: flex;
  gap: 15px;
  flex-wrap: wrap;       /* 小屏幕下换行 */
  flex: 1 1 auto;
}

.stats-row span {
  white-space: nowrap;   /* 成功、失败、跳过统计项不换行 */
}

/* 颜色保持你之前的设置 */
.success {
  color: #67c23a; /* 绿色 */
}

.failed {
  color: #f56c6c; /* 红色 */
}

.skipped {
  color: #e6a23c; /* 橙色，警告色 */
}
::v-deep(.el-dialog__body) {
  display: flex;
  flex-direction: column;
  align-items: center;       /* 水平居中 */
  justify-content: center;   /* 垂直居中 */
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
