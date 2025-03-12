<template>
  <div class="calendar-container" :class="{ expanded: isExpanded }">
    <!-- 切换按钮 -->
    <button class="toggle-btn" @click="toggleExpand">
      {{ isExpanded ? "◀ 收起" : "▶ 展开" }}
    </button>

    <!-- 日历面板始终显示 -->
    <div class="calendar-content">
      <h3>📅 选择时间段</h3>
      <el-date-picker
        v-model="dateRange"
        type="daterange"
        unlink-panels
        range-separator="至"
        start-placeholder="开始日期"
        end-placeholder="结束日期"
        format="YYYY-MM-DD"
        value-format="YYYY-MM-DD"
      />
      <button class="submit-btn" @click="sendToBackend">📤 发送到后端</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from "vue";
import { ElDatePicker } from "element-plus";

// 控制日历是否展开
const isExpanded = ref(false);
// 存储选择的日期范围
const dateRange = ref<[string, string] | null>(null);

// 切换日历的展开状态
const toggleExpand = () => {
  isExpanded.value = !isExpanded.value;
};

// 发送数据到后端
const sendToBackend = async () => {
  if (!dateRange.value || dateRange.value.length !== 2) {
    alert("❌ 请选择起始和结束日期！");
    return;
  }

  try {
    const response = await fetch("https://your-backend-api.com/submit", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        startDate: dateRange.value[0],
        endDate: dateRange.value[1]
      })
    });

    if (!response.ok) throw new Error(`HTTP 错误！状态码：${response.status}`);

    const result = await response.json();
    console.log("✅ 数据发送成功:", result);
    alert("✅ 时间段已提交！");
  } catch (error) {
    console.error("❌ 发送数据失败:", error);
    alert("❌ 发送失败，请检查网络！");
  }
};
</script>

<style scoped>
/* 日历容器的基础样式 */
.calendar-container {
  position: fixed;
  top: 100px;
  right: 0;
  width: 300px;
  background: #fff;
  box-shadow: -2px 0 10px rgba(0, 0, 0, 0.1);
  padding: 10px;
  border-radius: 8px 0 0 8px;
  transition: transform 0.3s ease-in-out;
  transform: translateX(100%);
}

/* 展开状态时，显示日历容器 */
.calendar-container.expanded {
  transform: translateX(0);
}

/* 切换按钮样式 */
.toggle-btn {
  position: absolute;
  left: -40px;
  top: 10px;
  padding: 8px 12px;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  z-index: 2;
}

/* 日历内容区域的样式 */
.calendar-content {
  padding: 10px;
  text-align: center;
}

/* 提交按钮样式 */
.submit-btn {
  margin-top: 10px;
  padding: 8px 12px;
  background: #28a745;
  color: white;
  border: none;
  cursor: pointer;
  border-radius: 4px;
}
</style>
