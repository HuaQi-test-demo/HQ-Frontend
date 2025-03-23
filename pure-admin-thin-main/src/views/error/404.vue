<template>
  <meta http-equiv="Access-Control-Allow-Origin" content="*" />
  <div>
    <div style="display: flex; width: 100%; align-items: center">
      <div class="selection">
        <div class="up">
          <CollapsibleCalendar
            class="select_calendar"
            @dateRangeSelected="handleDateRangeChange"
          />
          <el-select
            v-model="selectedCountries"
            multiple
            placeholder="请选择国家"
            class="select_country"
          >
            <el-option
              v-for="(countryZh, countryEn) in countryMapping"
              :key="countryEn"
              :label="countryZh"
              :value="countryEn"
              class="select_country"
            />
          </el-select>
        </div>
        <div class="down">
          <el-input
            v-model.number="maxDrawdown"
            type="number"
            placeholder="请输入最大回撤比例"
            class="drawdown-input"
          >
            <template #append>%</template>
          </el-input>
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
      </div>
      <el-button class="button" @click="updateCountries"> 更新国家 </el-button>
    </div>
    <div class="chart_container">
      <RiskChart class="chart1" :chartData="chartData1" />
      <RiskChart class="chart2" :chartData="chartData2" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, watch } from "vue";
import { useRouter } from "vue-router";
import RiskChart from "@/components/LineChart/RiskChart.vue";
import countryNameZhMapping from "@/json/mapping/country-name-zh-mapping.json";
import CollapsibleCalendar from "@/components/Calendar/CollapsibleCalendar.vue";
import axios from "axios";
import { ElMessage } from "element-plus";

// 定义 ChartData 类型
interface ChartData {
  time: string[];
  values: number[];
}

const router = useRouter();

// 响应式数据
const selectedCountries = ref<string[]>([]);
const dateRange = ref<[Date, Date]>([
  new Date("2024-12-01"),
  new Date("2024-12-07")
]); // 默认日期范围
const chartData1 = ref<ChartData>({ time: [], values: [] });
const chartData2 = ref<ChartData>({ time: [], values: [] });

// 国家映射
const countryMapping = ref<Record<string, string>>(countryNameZhMapping);

const maxDrawdown = ref<number>(2); // 最大回撤比例，默认为2%
const selectedPeriod = ref<number>(1); // 交易期限，默认为1年

// 初始化逻辑
onMounted(() => {
  const query = router.currentRoute.value.query;

  // 场景1：通过地图跳转带国家参数
  if (query.countries) {
    const countries = query.countries.toString().split(",");
    selectedCountries.value = countries;
    console.log("来自地图跳转的初始化参数:", {
      countries,
      dateRange: dateRange.value
    });
    fetchChartData(); // 立即请求数据
  }
});

// 处理日期范围变化
const handleDateRangeChange = (range: [Date, Date]) => {
  dateRange.value = range;
  console.log("新日期范围:", range);
  fetchChartData();
};

// 更新国家
const updateCountries = () => {
  console.log("更新国家:", selectedCountries.value);
  fetchChartData();
};

// 统一数据获取方法
const fetchChartData = async () => {
  if (selectedCountries.value.length !== 2) {
    ElMessage.warning("请选择两个国家");
    return;
  }
  if (maxDrawdown.value === null || maxDrawdown.value <= 0) {
    ElMessage.warning("请输入有效的最大回撤比例");
    return;
  }

  if (selectedPeriod.value === null) {
    ElMessage.warning("请选择交易期限");
    return;
  }
  console.log("开始请求 API...");
  try {
    console.log("发送请求参数:", {
      countries: selectedCountries.value,
      startDate: dateRange.value[0], //.toISOString().split("T")[0], // 转换为 YYYY-MM-DD 格式

      endDate: dateRange.value[1], //.toISOString().split("T")[0],
      maxDrawdown: maxDrawdown.value,
      investmentPeriod: selectedPeriod.value
    });

    const response = await axios.post(
      "http://121.36.9.36:5959/currency_pair/",
      {
        countries: selectedCountries.value,
        startDate: dateRange.value[0], //.toISOString().split("T")[0],
        endDate: dateRange.value[1], //.toISOString().split("T")[0],
        maxDrawdown: maxDrawdown.value,
        investmentPeriod: selectedPeriod.value
      }
    );
    console.log("API 响应:", response.data); // 🚀 确保 API 返回了数据
    // 处理响应数据
    chartData1.value = {
      time: response.data.data.date_time, // x 轴
      values: response.data.data.predict_rate.map(Number) // y 轴（转为 number）
    };

    chartData2.value = {
      time: response.data.data.date_time, // x 轴
      values: response.data.data.true_rate.map(Number) // y 轴（转为 number）
    };
  } catch (error) {
    console.error("请求失败:", error);
    ElMessage.error("数据加载失败");
  }
};

// 自动响应参数变化
watch([selectedCountries, dateRange], () => {
  if (selectedCountries.value.length === 2 && dateRange.value) {
    fetchChartData();
  }
});
</script>

<style scoped>
.max {
  width: 100%;
  display: flex;
  align-items: center;
}
.selection {
  width: 100%;
  display: block;
}

.up {
  display: flex;
}

.down {
  display: flex;
}

.select_calendar {
  padding: 10px;
  width: 100%;
}
.select_country {
  padding: 10px;
  width: 100%;
}

.drawdown-input {
  padding: 10px;
  width: 100%; /* max-width: 200px; */
}

.drawdown-input :deep(.el-input-group__append) {
  padding: 0 12px;
  background-color: var(--el-fill-color-light);
}

.period-select {
  padding: 10px;
  width: 100%;
}
.chart_container {
  margin-top: 50px;
  display: flex;
}
.button {
  margin-left: 20px;
  align-items: center;
  width: 10%;
}
.chart1 {
  width: 50%;
}
.chart2 {
  width: 50%;
}
</style>
