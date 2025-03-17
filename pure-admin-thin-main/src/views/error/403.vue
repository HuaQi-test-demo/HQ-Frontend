<script setup lang="ts">
import { ref, onMounted, nextTick } from "vue";
import * as echarts from "echarts";
import $ from "jquery";

// 🔥 创建两个 chartRef 分别绑定左侧 & 右侧的图表
const chartRefLeft = ref<HTMLDivElement | null>(null);
const chartRefRight = ref<HTMLDivElement | null>(null);

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
              repulsion: 100000,
              gravity: 0.02,
              edgeLength: [50, 300]
            },
            boundingRect: [-500, -500, 1000, 1000],
            data: json.nodes.map((node: any) => ({
              x: node.x,
              y: node.y,
              id: node.id,
              name: node.label,
              symbolSize: node.size,
              draggable: true,
              itemStyle: { color: node.color }
            })),
            edges: json.edges
              .filter((edge: any) => edge.size !== 0) // 过滤掉 size 为 0 的边
              .map((edge: any) => ({
                source: edge.sourceID,
                target: edge.targetID
              })),
            emphasis: {
              scale: 1.8, // 🔥 悬浮时放大 1.8 倍
              focus: "adjacency",
              label: {
                position: "right",
                show: true,
                fontSize: 20,
                color: "#ff5722"
              },
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
    initChart(chartRefLeft.value); // 🔥 初始化左侧图表
    initChart(chartRefRight.value); // 🔥 初始化右侧图表
  });
});
</script>

<template>
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
</template>

<style scoped>
.container {
  display: flex;
  height: 75vh;
}

/* 左侧 */
.left,
.right {
  flex: 1; /* 📌 左右平分 50% */
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
