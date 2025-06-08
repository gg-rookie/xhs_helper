<script setup>
import { ref, watch, onMounted, onBeforeUnmount } from 'vue'
import * as echarts from 'echarts/core'
import { CanvasRenderer } from 'echarts/renderers'
import 'echarts-liquidfill'

const props = defineProps({
  percent: {
    type: Number,
    required: true
  },
  status: {
    type: String,
    default: 'success' // success, warning, exception
  },
  size: {
    type: Number,
    default: 180
  }
})

const chartRef = ref(null)
let chartInstance = null

const getColorByStatus = (status) => {
  switch (status) {
    case 'success':
      return {
        main: '#67c23a',         // 明亮绿色
        wave: 'rgba(103, 194, 58, 0.6)',  // 透明绿色波浪
        shadow: 'rgba(103, 194, 58, 0.9)'
      }
    case 'warning':
      return {
        main: '#e6a23c',         // 明亮橙色
        wave: 'rgba(230, 162, 60, 0.6)',
        shadow: 'rgba(230, 162, 60, 0.9)'
      }
    case 'exception':
      return {
        main: '#f56c6c',         // 明亮红色
        wave: 'rgba(245, 108, 108, 0.6)',
        shadow: 'rgba(245, 108, 108, 0.9)'
      }
    default:
      return {
        main: '#409eff',
        wave: 'rgba(64, 158, 255, 0.6)',
        shadow: 'rgba(64, 158, 255, 0.9)'
      }
  }
}

const initChart = () => {
  if (!chartRef.value) return
  chartInstance = echarts.init(chartRef.value)
  setChartOptions()
}

const setChartOptions = () => {
  if (!chartInstance) return
  const color = getColorByStatus(props.status)
  const percent = Math.min(Math.max(props.percent, 0), 100) / 100

  const option = {
    series: [
      {
        type: 'liquidFill',
        radius: '90%',
        data: [percent],
        waveLength: '100%',
        waveAmplitude: '8%',
        outline: {
          show: true,
          borderDistance: 6,
          itemStyle: {
            borderColor: color.main,
            borderWidth: 5,
          }
        },
        backgroundStyle: {
          color: '#e4e7ed',
        },
        itemStyle: {
          color: color.main,
          shadowBlur: 20,
          shadowColor: color.shadow,
        },
        emphasis: {
          itemStyle: {
            color: color.wave,
          }
        },
        label: {
          formatter: `${props.percent}%`,
          fontSize: props.size / 4,
          color: color.main,
          fontWeight: 'bold',
        },
        waveAnimation: true,
      }
    ]
  }
  chartInstance.setOption(option)
}

watch(
  () => [props.percent, props.status, props.size],
  () => {
    setChartOptions()
  }
)

onMounted(() => {
  initChart()
  window.addEventListener('resize', () => {
    chartInstance && chartInstance.resize()
  })
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', () => {
    chartInstance && chartInstance.resize()
  })
  chartInstance && chartInstance.dispose()
})
</script>

<template>
  <div
    ref="chartRef"
    :style="{ width: size + 'px', height: size + 'px' }"
  ></div>
</template>
