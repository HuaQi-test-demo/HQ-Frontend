<script setup lang="ts">
import { ref, onMounted, nextTick, watch } from "vue";
import axios from "axios";
import * as echarts from "echarts";
import $ from "jquery";
import { useRouter } from "vue-router";
import { ElMessage } from "element-plus";

interface ChartData {
  time: string[];
  values: number[];
}
const router = useRouter();

const chartRefLeft = ref<HTMLDivElement | null>(null);
const chartRefRight = ref<HTMLDivElement | null>(null);
const maxDrawdown = ref<number>(2); // 最大回撤比例，默认为2%
const selectedMonth = ref<number>(null); // 单选月份

// 📌 复用 ECharts 初始化函数
const initChart = (chartRef: HTMLDivElement | null) => {
  if (!chartRef) {
    console.error("❌ chartRef 未绑定到 DOM！");
    return;
  }

  const myChart = echarts.init(chartRef, null, { renderer: "svg" });
  myChart.showLoading();

  $.getJSON("npmdepgraph.min10.json")
    .done(json => {
      myChart.hideLoading();
      const option = {
        animationDurationUpdate: 1000,
        animationEasingUpdate: "elasticOut" as any,
        series: [
          {
            type: "graph",
            roam: true,
            layout: "force",
            force: {
              repulsion: 1,
              gravity: 0.02,
              edgeLength: [50, 2000]
            },
            boundingRect: [-500, -500, 1000, 1000],
            data: json.nodes.map((node: any) => ({
              x: node.x,
              y: node.y,
              id: node.id,
              name: node.label,
              symbolSize: node.size,
              draggable: true,
              itemStyle: { color: node.color },
              label: { show: true } // 默认不显示 label
            })),
            edges: json.edges
              .filter((edge: any) => edge.size !== 0) // 过滤掉 size 为 0 的边
              .map((edge: any) => ({
                source: edge.sourceID,
                target: edge.targetID
              })),
            emphasis: {
              scale: 1.2,
              focus: "adjacency",
              roam: false,
              lineStyle: { width: 0.5, curveness: 0.3, opacity: 0.7 }
            }
          }
        ]
      };
      myChart.setOption(option);
    })
    .fail((jqxhr, textStatus, error) => {
      console.error("❌ 请求 JSON 失败：", textStatus, error);
    });
  myChart.on("graphRoam", function () {
    const updatedOption = myChart.getOption();
    const graph = updatedOption.series[0];
    graph.data.forEach((node: any) => {
      node.x = Math.max(-500, Math.min(500, node.x));
      node.y = Math.max(-500, Math.min(500, node.y));
    });

    myChart.setOption(updatedOption);
  });
};

onMounted(() => {
  nextTick(() => {
    initChart(chartRefLeft.value);
    initChart(chartRefRight.value);
    nextTick(fetchChartData);
  });
});

const fetchChartData = async () => {
  if (selectedMonth.value === null) {
    ElMessage.warning("请选择一个月份");
    return;
  }
  if (maxDrawdown.value === null || maxDrawdown.value <= 0) {
    ElMessage.warning("请输入有效的最大回撤比例");
    return;
  }
  try {
    console.log("发送请求参数:", {
      month: selectedMonth.value, // 🔹 单选
      maxDrawdown: maxDrawdown.value
    });
    const response = await axios.post(
      "http://121.36.9.36:5959/multi_currency/",
      {
        month: selectedMonth.value, // 🔹 直接传单个值
        maxDrawdown: maxDrawdown.value
      }
    );
    if (response.data.result1 && response.data.result2) {
      console.log(
        "📊 接收到的 result 数据:",
        response.data.result1,
        response.data.result2
      );
      updateCharts(response.data.result1, response.data.result2); // 更新图表
    } else {
      console.error("⚠️ 后端返回的数据格式不正确:", response.data);
      ElMessage.error("数据格式错误，请检查后端返回内容");
    }
  } catch (error) {
    console.error("❌ 请求失败:", error);
    ElMessage.error("数据加载失败");
  }
};
const updateCharts = (result1: any, result2: any) => {
  if (!result1.nodes || !result1.edges || !result2.nodes || !result2.edges) {
    console.error("⚠️ result 数据缺少必要字段:", { result1, result2 });
    ElMessage.error("数据不完整，无法更新图表");
    return;
  }

  const leftChart = echarts.getInstanceByDom(chartRefLeft.value!);
  const rightChart = echarts.getInstanceByDom(chartRefRight.value!);

  if (!leftChart || !rightChart) {
    console.error("❌ ECharts 实例未初始化");
    return;
  }

  const formatGraphData = (data: any) => ({
    animationDurationUpdate: 1000,
    animationEasingUpdate: "elasticOut" as any,
    series: [
      {
        type: "graph",
        layout: "force",
        roam: true,
        force: {
          repulsion: 1,
          gravity: 0.02,
          edgeLength: [50, 2000]
        },
        boundingRect: [-500, -500, 1000, 1000],
        data: data.nodes.map((node: any) => ({
          x: node.x,
          y: node.y,
          id: node.id,
          name: node.label,
          symbolSize: node.size,
          draggable: true,
          itemStyle: { color: node.color },
          label: { show: true }
        })),
        edges: data.edges.map((edge: any) => ({
          source: edge.sourceID,
          target: edge.targetID
        })),
        emphasis: {
          scale: 1.2,
          focus: "adjacency",
          roam: false,
          lineStyle: { width: 0.5, curveness: 0.3, opacity: 0.7 }
        }
      }
    ]
  });

  // 更新左侧图表
  leftChart.setOption(formatGraphData(result1));

  // 更新右侧图表
  rightChart.setOption(formatGraphData(result2));
};
</script>

<template>
  <div class="max">
    <div class="up">
      <el-input
        v-model.number="maxDrawdown"
        type="number"
        placeholder="请输入最大回撤比例"
        class="drawdown-input"
      />
      <!-- 📌 月份选择框 -->
      <el-select
        v-model="selectedMonth"
        placeholder="请选择月份"
        class="select-month"
      >
        <el-option
          v-for="month in 12"
          :key="month"
          :label="`${month} 月`"
          :value="`2024-${month.toString().padStart(2, '0')}-01`"
        />
      </el-select>

      <el-button class="button" @click="fetchChartData">更新数据</el-button>
    </div>
    <div class="container">
      <div class="left">
        <h2>多货币回撤图示</h2>
        <br />
        <div ref="chartRefLeft" class="chart" />
      </div>
      <div class="right">
        <h2>对比图</h2>
        <br />
        <div ref="chartRefRight" class="chart" />
      </div>
    </div>
  </div>
</template>
<style scoped>
.max {
  display: block;
}
.up {
  display: flex;
}
.drawdown-input {
  margin: 20px;
  width: 100%;
}
.select-month {
  margin: 20px;
  width: 100%;
}
.button {
  margin: 20px;
  width: 20%;
}
.container {
  display: flex;
  height: 75vh;
}

/* 左侧 */
.left,
.right {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 10px;
}

/* 图表区域 */
.chart {
  height: 600px;
  width: 100%;
  background-color: #f9f9f9;
}
</style>
