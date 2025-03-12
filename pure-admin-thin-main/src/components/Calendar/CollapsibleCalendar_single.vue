<template>
  <div class="calendar-container">
    <input
      type="date"
      v-model="selectedDate"
      @change="handleDateChange"
      class="date-picker"
    />
  </div>
</template>

<script lang="ts" setup>
import { ref, defineEmits } from "vue";

const emit = defineEmits(["dateSelected"]);
const selectedDate = ref("");

const handleDateChange = () => {
  if (selectedDate.value) {
    emit("dateSelected", selectedDate.value);
  }
};
</script>

<style scoped>
.calendar-container {
  padding: 10px;
  text-align: center;
  margin: 10px;
  width: 150px;
}
.date-picker {
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 16px;
}
</style>
