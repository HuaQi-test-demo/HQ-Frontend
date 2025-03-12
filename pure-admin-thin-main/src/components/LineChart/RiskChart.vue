<template>
  <div class="chart-container">
    <v-chart class="chart" :option="chartOption" autoresize />
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from "vue";
import { use } from "echarts/core";
import VChart from "vue-echarts";
import { LineChart } from "echarts/charts";
import {
  GridComponent,
  TooltipComponent,
  LegendComponent
} from "echarts/components";
import { CanvasRenderer } from "echarts/renderers";

// 注册 ECharts 组件
use([
  GridComponent,
  TooltipComponent,
  LegendComponent,
  LineChart,
  CanvasRenderer
]);

// 模拟风险信号数据
const riskData = ref([
  { time: "10:00", value: 45 },
  { time: "10:10", value: 48 },
  { time: "10:20", value: 55 }, // 超阈值
  { time: "10:30", value: 60 }, // 超阈值
  { time: "10:40", value: 52 }, // 超阈值
  { time: "10:50", value: 47 },
  { time: "11:00", value: 43 }
]);

const threshold = ref(50); // 设定阈值

// 计算 ECharts 配置
const chartOption = computed(() => {
  return {
    title: {
      text: "风险信号波动图",
      left: "center"
    },
    tooltip: {
      trigger: "axis",
      formatter: (params: any) => {
        const { name, value } = params[0];
        return `时间: ${name} <br> 值: ${value}`;
      }
    },
    xAxis: {
      type: "category",
      data: riskData.value.map(item => item.time),
      boundaryGap: false
    },
    yAxis: {
      type: "value"
    },
    series: [
      {
        name: "风险信号",
        type: "line",
        data: riskData.value.map(item => item.value),
        smooth: true,
        lineStyle: {
          width: 2
        },
        markArea: {
          itemStyle: {
            color: "rgba(255, 0, 0, 0.2)" // 红色高亮超阈值区间
          },
          data: generateThresholdAreas(riskData.value, threshold.value)
        }
      }
    ]
  };
});

// 生成超阈值区间
function generateThresholdAreas(
  data: { time: string; value: number }[],
  threshold: number
) {
  let areas = [];
  let startIndex = -1;

  for (let i = 0; i < data.length; i++) {
    if (data[i].value > threshold && startIndex === -1) {
      startIndex = i;
    } else if (data[i].value <= threshold && startIndex !== -1) {
      areas.push([
        { xAxis: data[startIndex].time },
        { xAxis: data[i - 1].time }
      ]);
      startIndex = -1;
    }
  }

  // 如果超阈值区间延续到最后
  if (startIndex !== -1) {
    areas.push([
      { xAxis: data[startIndex].time },
      { xAxis: data[data.length - 1].time }
    ]);
  }

  return areas;
}
</script>

<style scoped>
.chart-container {
  width: 100%;
  height: 400px;
}
.chart {
  width: 100%;
  height: 100%;
}
</style>
