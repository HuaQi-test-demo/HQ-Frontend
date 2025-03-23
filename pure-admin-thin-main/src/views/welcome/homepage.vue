<!-- eslint-disable vue/no-parsing-error -->
<!-- eslint-disable prettier/prettier -->
<template>
  <div class="dashboard">
      
    <!-- 中间列 -->
    <div class="column2">
      <el-date-picker
        v-model="selectedDate"
        type="date"
        placeholder="选择日期"
        :disabled-date="disabledDate"
        @change="handleDateSelected"
      />
      <div ref="chartRef" class="chart" :style="chartStyle"/>
      <div v-if="showExchangeRate" class="chart-container exchange-rate-container">
        <div class="exchange-rate-details">
          <div class="header">
            <h2>{{ selectedCountry }} - {{ selectedDate }} 汇率详情</h2>
            <button class="close-button" @click="closeExchangeRate">×</button>
          </div>
          <table id="exchangeRatesTable">
            <thead>
              <tr>
                <th rowspan="2">货币</th>
                <th rowspan="2">货币符号</th>
                <th colspan="2">主流货币信息</th>
              </tr>
              <tr>
                <th>货币名称</th>
                <th>货币汇率</th>
              </tr>
            </thead>
            <tbody v-if="new_data">
              <tr>
                <td rowspan="9" style="vertical-align: middle">
                  {{ new_data.currency }}
                </td>
                <td rowspan="9" style="vertical-align: middle">
                  {{ new_data.currency_sign }}
                </td>
                <td>{{ new_data[`currency1_name`] }}</td>
                <td>{{ new_data[`currency_rate1`] }}</td>
              </tr>
              <tr v-for="index in 8" :key="index">
                <td>{{ new_data[`currency${index + 1}_name`] || "---" }}</td>
                <td>{{ new_data[`currency_rate${index + 1}`] || "---" }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
    <div class="column1">
      <div ref="pieChart" class="chart-container"/>
      <div ref="lineChart" class="chart-container"/>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted, nextTick, computed, watch } from "vue";
import * as echarts from "echarts";
import geo from "@/json/geo";
import mapping from "@/json/mapping";
import CollapsibleCalendar from "@/components/Calendar/CollapsibleCalendar_single.vue";
import { useRouter } from "vue-router";
import axios from "axios";

const router = useRouter();
const chartRef = ref<HTMLDivElement | null>(null);
const selectedCountries = ref<string[]>([]);
const selectedDate = ref<string>("2024-01-01");
const selectedCountry = ref<string>("");
const showExchangeRate = ref<boolean>(false);
const myChart = ref<echarts.ECharts | null>(null); // 保存 ECharts 实例
// 主要货币数据
const currencyData = [
  {
    name: "EUR",
    rates: [
      0.92265, 0.92665, 0.9333, 0.92325, 0.93335, 0.9246, 0.9052, 0.89535,
      0.92095, 0.94545, 0.96095
    ]
  },
  {
    name: "JPY",
    rates: [
      147.365, 150.655, 151.345, 156.32, 156.985, 160.88, 152.7, 146.18,
      142.815, 153.245, 149.755, 157
    ]
  },
  {
    name: "GBP",
    rates: [
      0.789827, 0.792111, 0.796242, 0.785577, 0.790826, 0.779029, 0.761789,
      0.747245, 0.771694, 0.785238, 0.796908
    ]
  },
  {
    name: "CNY",
    rates: [
      7.198, 7.22235, 7.229, 7.23275, 7.26725, 7.241, 7.0913, 7.01105, 7.11615,
      7.24605, 7.29935
    ]
  }
];
// 地图样式
const chartStyle = ref({ width: "100%", height: "500px" }); // 确保这个高度合适
const pieChart = ref<HTMLDivElement | null>(null);

// 设定可选范围，比如只允许 2024 年内选择
const disabledDate = date => {
  const minDate = new Date("2024-01-01");
  const maxDate = new Date("2024-12-31");
  return date < minDate || date > maxDate;
};

// 监听 showExchangeRate 的变化
watch(showExchangeRate, () => {
  // 在下一个 DOM 更新周期后调用 resize
  nextTick(() => {
    if (myChart.value) {
      myChart.value.resize(); // 手动调整 ECharts 尺寸
    }
  });
});
// 发送数据到后端
const new_data = ref<any>(null);
const sendToBackend = async (datas: {
  country?: string;
  date?: string;
  countries?: string[];
}) => {
  try {
    const response = await axios.post("http://121.36.9.36:5959/submit/", datas);
    new_data.value = response.data.table_data;
    console.log(new_data);
  } catch (error) {
    console.error("Error sending data to backend:", error);
    // 提示用户后端连接失败，但地图应该仍然显示
    alert("无法连接到后端，地图将加载默认数据。");
  }
};
console.log(new_data);
onMounted(async () => {
  await nextTick(); // 确保 DOM 完成渲染
  if (!chartRef.value) {
    console.error("❌ chartRef 未绑定到 DOM！");
    return;
  }
  myChart.value = echarts.init(chartRef.value);
  if (!myChart.value) {
    console.error("❌ ECharts 初始化失败！");
    return;
  }
  echarts.registerMap("WorldCountry", geo.WorldCountryGeo);
  const options: echarts.EChartsOption = {
    tooltip: { trigger: "item" },
    toolbox: {
      show: true,
      orient: "vertical",
      left: "right",
      top: "center",
      feature: {
        dataView: { readOnly: false },
        restore: {},
        saveAsImage: {}
      }
    },
    visualMap: {
      min: 0,
      max: 1,
      show: false,
      inRange: { color: ["rgb(078,101,155)", "rgb(183, 118, 108)"] }
    },
    series: [
      {
        name: "世界地图",
        type: "map",
        map: "WorldCountry",
        label: { show: false },
        itemStyle: {
          areaColor: "rgb(078,101,155)",
          borderColor: "rgba(253, 207, 158, 0.6)",
          borderWidth: 0.3
        },
        emphasis: {
          label: { show: true, fontSize: "14" },
          itemStyle: {
            areaColor: "rgb(183, 118, 108)",
            borderColor: "rgba(253, 207, 158, 0.8)",
            borderWidth: 0.8
          }
        },
        nameMap: mapping.CountryNameZhMapping
      }
    ]
  };
  myChart.value.setOption(options);
  myChart.value.on("click", (params: any) => {
    if (!params.name) return;
    const country = params.name;
    if (
      !selectedCountries.value.includes(country) &&
      selectedCountries.value.length < 2
    ) {
      selectedCountries.value.push(country);
      selectedCountry.value = country;
    }
    // 如果已经选择了日期，发送国家和日期到后端
    if (selectedDate.value) {
      sendToBackend({
        country: selectedCountry.value,
        date: selectedDate.value
      });
    }

    if (selectedCountries.value.length === 2) {
      console.log("选中的两个国家:", selectedCountries.value);
      // 发送选中的两个国家到后端
      sendToBackend({ countries: selectedCountries.value }).then(() => {
        router.push({
          path: "/404", // 跳转至单货币对详情页
          query: { countries: selectedCountries.value.join(",") }
        });
        selectedCountries.value = [];
      });
    } else if (selectedDate.value) {
      console.log(
        "选中的国家和日期:",
        selectedCountry.value,
        selectedDate.value
      );
      showExchangeRate.value = true;
    }
  });
  console.log(myChart.value.getOption());
  window.addEventListener("resize", () => myChart.value?.resize());

  // 柱状图
  const lineChart = echarts.init(
    document.querySelectorAll(".chart-container")[1] as any
  );

  lineChart.setOption({
    title: { text: "📈 主要货币汇率趋势", top: 0 },
    tooltip: { trigger: "axis" },
    legend: { data: currencyData.map(c => c.name), top: 30 },
    grid: { top: 60, left: "10%", right: "10%", bottom: "10%" },
    xAxis: {
      type: "category",
      data: Array.from(
        { length: currencyData[0].rates.length },
        (_, i) => `Day ${i + 1}`
      )
    },
    yAxis: { type: "value" },
    series: currencyData.map(currency => ({
      name: currency.name,
      type: "line",
      data: currency.rates
    }))
  });

  // 饼图（市场占比）
  if (!pieChart.value) {
    console.error("❌ pieChart 未绑定到 DOM！");
    return;
  }

  // 绑定 ECharts
  const pieChartInstance = echarts.init(pieChart.value);
  pieChartInstance.setOption({
    title: { text: "🍕 主要货币占比" },
    tooltip: { trigger: "item" },
    series: [
      {
        type: "pie",
        data: [
          { name: "USD", value: 47.89 },
          { name: "EUR", value: 22.85 },
          { name: "JPY", value: 3.61 },
          { name: "GBP", value: 6.84 },
          { name: "CNY", value: 4.47 },
          { name: "other", value: 14.34 }
        ]
      }
    ]
  });
});

const handleDateSelected = (date: string) => {
  selectedDate.value = date;
  if (selectedCountry.value) {
    console.log("选中的国家和日期:", selectedCountry.value, selectedDate.value);
    showExchangeRate.value = true;
  }
  // 发送国家和日期到后端
  sendToBackend({ country: selectedCountry.value, date: selectedDate.value });
};

// 关闭汇率详情
const closeExchangeRate = () => {
  showExchangeRate.value = false;
  selectedCountries.value = [];
  selectedDate.value = "";
  selectedCountry.value = "";
};
</script>

<style scoped>
.dashboard {
  display: flex;
  justify-content: space-between;
  padding: 20px;
  gap: 5px;
}
.column1 {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.column2 {
  flex: 2;
  display: flex;
  flex-direction: column;
  gap: clamp(5px, 2%, 30px); /* 动态控制 gap */
  height: 100%; /* 确保父元素有明确高度 */
}
.center {
  align-items: center;
}
.chart-container {
  flex: 1; /* 让子元素平分父元素高度 */
  width: 100%;
  min-height: 0; /* 解决 flex 子元素塌陷问题 */
}

.calendar {
  display: flex;
  justify-content: flex-start; /* 左对齐 */
  width: 100%;
  height: 30px;
}

.exchange-rate-container {
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
}

.exchange-rate-details {
  width: 100%;
  height: 100%;
  padding: 20px;
  border-radius: 8px;
  text-align: center;
}

.header {
  display: flex;
  justify-content: center;
  /* 标题居中 */
  align-items: center;
  position: relative;
  /* 为按钮定位提供参考 */
  margin-bottom: 16px;
}

.close-button {
  position: absolute;
  /* 绝对定位 */
  right: 0;
  /* 放置在最右侧 */
  width: 28px;
  height: 28px;
  cursor: pointer;
  background: rgb(215, 213, 213);
  padding: 2px;
  border-radius: 8px;
  font-weight: bold;
  color: black;
  font-size: 18px;
  transition: background-color 0.3s ease;
  border: none;
  outline: none;
}

.close-button:hover {
  background-color: rgb(194, 191, 191);
}

#exchangeRatesTable {
  width: 100%;
  border-collapse: collapse;
  margin-top: 16px;
}

#exchangeRatesTable th,
#exchangeRatesTable td {
  padding: 12px;
  border: 1px solid #ded3d3;
  text-align: center;
}

#exchangeRatesTable th {
  background-color: #f5f5f5;
  font-weight: bold;
}

#exchangeRatesTable tr:hover {
  background-color: #f9f9f9;
}
</style>
