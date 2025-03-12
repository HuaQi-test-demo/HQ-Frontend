<template>
  <div>
    <RiskChart />
    <div class="flex justify-center items-center h-[640px]">
      <div class="ml-12">
        <el-button
          v-motion
          type="primary"
          :initial="{
            opacity: 0,
            y: 100
          }"
          :enter="{
            opacity: 1,
            y: 0,
            transition: {
              delay: 160
            }
          }"
          @click="router.push('/')"
        >
          返回首页
        </el-button>
      </div>
    </div>
    <div class="country-selection">
      <el-select
        v-model="selectedCountries"
        multiple
        placeholder="请选择国家"
        style="width: 300px; margin-top: 20px"
      >
        <el-option
          v-for="(countryZh, countryEn) in countryMapping"
          :key="countryEn"
          :label="countryZh"
          :value="countryEn"
        />
      </el-select>
      <el-button @click="updateCountries" style="margin-left: 10px"
        >更新国家</el-button
      >
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue";
import { useRouter } from "vue-router";
import RiskChart from "@/components/LineChart/RiskChart.vue";
import countryNameZhMapping from "@/json/mapping/country-name-zh-mapping.json";

const router = useRouter();
const countryMapping = ref<Record<string, string>>(countryNameZhMapping);
const selectedCountries = ref<string[]>([]);

onMounted(() => {
  const query = router.currentRoute.value.query;
  if (query.countries) {
    // 初始化选中的国家
    selectedCountries.value = query.countries.toString().split(",");
    console.log("初始化选中的国家:", selectedCountries.value);
  }
});

const updateCountries = () => {
  console.log("更新后的国家:", selectedCountries.value);
  // 这里可以调用API将更新后的国家传递给后端
};
</script>

<style scoped>
.country-selection {
  margin-top: 20px;
  text-align: center;
}
</style>
