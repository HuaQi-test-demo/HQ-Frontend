<script setup lang="ts">
import { ref, onMounted, nextTick, watch, computed } from "vue";
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
const maxDrawdown = ref<number>(-0.1); // 最大回撤比例，默认为-0.1
const selectedMonth = ref<number>(12); // 单选月份
const selectedPeriod = ref<number>(1); // 交易期限，默认为1年
const availableMonths = computed(() => {
  if (selectedPeriod.value === 1) {
    return [10, 11, 12];
  } else if (selectedPeriod.value === 3) {
    return [5, 6, 7, 8, 9, 10, 11, 12];
  } else {
    return Array.from({ length: 12 }, (_, i) => i + 1);
  }
});
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
const validateInput = () => {
  if (maxDrawdown.value >= 0) {
    maxDrawdown.value = -Math.abs(maxDrawdown.value);
  }
};
const fetchChartData = async () => {
  if (selectedMonth.value === null) {
    ElMessage.warning("请选择一个月份");
    return;
  }
  if (maxDrawdown.value === null || maxDrawdown.value > 0) {
    ElMessage.warning("请输入有效的最大回撤比例");
    return;
  }
  if (selectedPeriod.value === null) {
    ElMessage.warning("请选择交易期限");
    return;
  }
  try {
    console.log("发送请求参数:", {
      month: selectedMonth.value, // 🔹 单选
      maxDrawdown: maxDrawdown.value,
      investmentPeriod: selectedPeriod.value
    });
    const response = await axios.post(
      "http://121.36.9.36:5959/multi_currency/",
      {
        month: selectedMonth.value, // 🔹 直接传单个值
        maxDrawdown: maxDrawdown.value,
        investmentPeriod: selectedPeriod.value
      }
    );
    if (response.data.result1 && response.data.result2) {
      console.log(
        "接收到的 result 数据:",
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

  const formatGraphData = (data: any) => {
    // 获取所有节点连接的边的 size
    const validEdges = data.edges;
    const validNodeIds = new Set(
      validEdges.flatMap((edge: any) => [edge.sourceID, edge.targetID])
    );

    return {
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
          // 过滤掉没有有效连接的节点
          data: data.nodes
            .filter((node: any) => validNodeIds.has(node.id))
            .map((node: any) => ({
              x: node.x,
              y: node.y,
              id: node.id,
              name: node.label,
              symbolSize: node.size,
              draggable: true,
              itemStyle: { color: node.color },
              label: { show: true }
            })),
          // 只保留 size 不为 0 的边
          edges: validEdges.map((edge: any) => {
            let color = "#CFD8DC"; // 默认颜色
            if (edge.size >= 1.3) {
              color = "#F44336";
            } else if (edge.size >= 1.1) {
              color = "#EF9A9A";
            } else if (edge.size >= 0.9) {
              color = "#CFD8DC";
            } else if (edge.size >= 0.7) {
              color = "#81D4FA";
            } else {
              color = "#03A9F4";
            }

            return {
              source: edge.sourceID,
              target: edge.targetID,
              lineStyle: { color }
            };
          }),
          emphasis: {
            scale: 1.2,
            focus: "adjacency",
            roam: false,
            lineStyle: { width: 0.5, curveness: 0.3, opacity: 0.7 }
          }
        }
      ]
    };
  };

  // 更新左侧图表
  leftChart.setOption(formatGraphData(result1));

  // 更新右侧图表
  rightChart.setOption(formatGraphData(result2));
};
</script>

<template>
  <div class="max">
    <div class="up">
      <div class="input1">
        <p style="margin-left: 20px">请输入最大回撤比例：</p>
        <el-input
          v-model.number="maxDrawdown"
          type="number"
          placeholder="请输入最大回撤比例"
          class="drawdown-input"
          :min="-100"
          :step="0.01"
          @change="validateInput"
        />
      </div>
      <div class="input1">
        <p style="margin-left: 20px">请选择交易期限：</p>
        <el-select
          v-model="selectedPeriod"
          placeholder="选择交易期限"
          class="period-select"
        >
          <el-option label="1年" :value="1" />
          <el-option label="3年" :value="3" />
          <el-option label="5年" :value="5" />
        </el-select>
      </div>
      <!--  月份选择框 -->
      <div class="input1">
        <p style="margin-left: 20px">请选择查看的月份：</p>
        <el-select
          v-model="selectedMonth"
          placeholder="请选择月份"
          class="select-month"
        >
          <el-option
            v-for="month in availableMonths"
            :key="month"
            :label="`${month} 月`"
            :value="`2024-${month.toString().padStart(2, '0')}-01`"
          />
        </el-select>
      </div>
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
.input1 {
  width: 100%;
  margin: 20px;
}
.drawdown-input {
  margin: 10px;
  width: 100%;
}

.period-select {
  margin: 10px;
  width: 100%;
}

.select-month {
  margin: 10px;
  width: 100%;
}

.button {
  margin: 45px;
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
