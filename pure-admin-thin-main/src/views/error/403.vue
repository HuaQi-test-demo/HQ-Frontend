<script setup lang="ts">
import CollapsibleCalendar from "@/components/Calendar/CollapsibleCalendar.vue";
import { useRouter } from "vue-router";

defineOptions({
  name: "403"
});

const router = useRouter();
</script>

<template>
  <div>
    <CollapsibleCalendar />
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
  </div>
</template>
