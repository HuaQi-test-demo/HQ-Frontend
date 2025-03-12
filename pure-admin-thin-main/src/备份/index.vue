<template>
  <div>
    <div ref="chartRef" class="chart"></div>
    <CollapsibleCalendar @dateSelected="handleDateSelected" />
    <div v-if="showExchangeRate" class="exchange-rate-container">
      <div class="exchange-rate-details">
        <h2>{{ selectedCountry }} - {{ selectedDate }} 汇率详情</h2>
        <p>汇率数据展示...</p>
        <button @click="closeExchangeRate">关闭</button>
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted, nextTick } from "vue";
import * as echarts from "echarts";
import geo from "@/json/geo";
import mapping from "@/json/mapping";
import CollapsibleCalendar from "@/components/Calendar/CollapsibleCalendar_single.vue";
import { useRouter } from "vue-router";

const router = useRouter();
const chartRef = ref<HTMLDivElement | null>(null);
const selectedCountries = ref<string[]>([]);
const selectedDate = ref<string>("");
const selectedCountry = ref<string>("");
const showExchangeRate = ref<boolean>(false);

onMounted(async () => {
  await nextTick();

  if (!chartRef.value) {
    console.error("❌ chartRef 未绑定到 DOM！");
    return;
  }

  const myChart = echarts.init(chartRef.value);
  if (!myChart) {
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
      inRange: { color: ["#d1e5f0", "#f46d43"] }
    },
    series: [
      {
        name: "世界地图",
        type: "map",
        map: "WorldCountry",
        label: { show: false },
        emphasis: {
          label: { show: true, fontSize: "14" },
          itemStyle: { areaColor: "#f46d43" }
        },
        nameMap: mapping.CountryNameZhMapping
      }
    ]
  };

  myChart.setOption(options);

  myChart.on("click", (params: any) => {
    if (!params.name) return;

    const country = params.name;
    if (selectedCountries.value.length < 2) {
      selectedCountries.value.push(country);
      selectedCountry.value = country;
    }

    if (selectedCountries.value.length === 2) {
      console.log("选中的两个国家:", selectedCountries.value);
      router.push({ path: "/404", query: { countries: selectedCountries.value.join(",") } });
    } else if (selectedDate.value) {
      console.log("选中的国家和日期:", selectedCountry.value, selectedDate.value);
      showExchangeRate.value = true;
    }
  });

  window.addEventListener("resize", () => myChart.resize());
});

const handleDateSelected = (date: string) => {
  selectedDate.value = date;
  if (selectedCountry.value) {
    console.log("选中的国家和日期:", selectedCountry.value, selectedDate.value);
    showExchangeRate.value = true;
  }
};

const closeExchangeRate = () => {
  showExchangeRate.value = false;
  selectedCountries.value = [];
  selectedDate.value = "";
  selectedCountry.value = "";
};
</script>

<style scoped>
.chart {
  width: 100%;
  height: 500px;
}

.exchange-rate-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

.exchange-rate-details {
  background: white;
  padding: 20px;
  border-radius: 8px;
  text-align: center;
}

</style>