<template>
  <div>
    <div class="country-selection">
       <div class="select_container">
      <CollapsibleCalendar @dateRangeSelected="handleDateRangeChange" class="select_calendar"/>
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
      <el-button class="button" @click="updateCountries">
        更新国家
      </el-button>
    </div>
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
const dateRange = ref<[Date, Date]>([new Date("2024-12-01"), new Date("2024-12-07")]); // 默认日期范围
const chartData1 = ref<ChartData>({ time: [], values: [] });
const chartData2 = ref<ChartData>({ time: [], values: [] });

// 国家映射
const countryMapping = ref<Record<string, string>>(countryNameZhMapping);

// 初始化逻辑
onMounted(() => {
  const query = router.currentRoute.value.query;

  // 场景1：通过地图跳转带国家参数
  if (query.countries) {
    const countries = query.countries.toString().split(",");
    selectedCountries.value = countries;
    console.log("来自地图跳转的初始化参数:", {
      countries,
      dateRange: dateRange.value,
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

  try {
    console.log("发送请求参数:", {
      countries: selectedCountries.value,
      startDate: dateRange.value[0].toISOString().split('T')[0], // 转换为 YYYY-MM-DD 格式
      endDate: dateRange.value[1].toISOString().split('T')[0], // 转换为 YYYY-MM-DD 格式
    });

    // 需替换实际API
    const response = await axios.post("/api/currency-pair", {
      countries: selectedCountries.value,
      startDate: dateRange.value[0].toISOString().split('T')[0],
      endDate: dateRange.value[1].toISOString().split('T')[0],
    });

    // 处理响应数据（假设返回结构）
    chartData1.value = {
      time: response.data.country1.dates,
      values: response.data.country1.rates,
    };
    chartData2.value = {
      time: response.data.country2.dates,
      values: response.data.country2.rates,
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
.country-selection {
  padding: 10px;
  text-align: center;
}
.select_container {
  display: flex;
  width: 100%;
}
.select_calendar {
  width: 100%;
}
.select_country {
  width: 100%;
  padding: 20px;
}
.chart_container {
  margin-top: 150px;
  display: flex;
}
.button {
  margin: 20px;
  width: 10%;
}
.chart1 {
  width: 50%;
}
.chart2 {
  width: 50%;
}
</style>