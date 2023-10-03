<template>
  <el-tabs
    v-model="editableTabsValue"
    type="card"
    editable
    class="demo-tabs"
    @edit="handleTabsEdit"
  >
    <el-tab-pane
      v-for="item in editableTabs"
      :key="item.name"
      :label="item.title"
      :name="item.name"
    >
      <MapGenerator :key="item.name" :index="item.name" />
    </el-tab-pane>
  </el-tabs>
</template>
<script setup>
import { ref } from "vue";
import MapGenerator from "@/components/MapGenerator.vue";
let tabIndex = 2;
const editableTabsValue = ref("1");
const editableTabs = ref([
  {
    title: "版式1",
    name: "1",
  },
  {
    title: "版式2",
    name: "2",
  },
]);
const handleTabsEdit = (targetName, action) => {
  if (action === "add") {
    const newTabName = `${++tabIndex}`;
    editableTabs.value.push({
      title: "版式" + newTabName,
      name: newTabName,
      content: "New Tab content",
    });
    editableTabsValue.value = newTabName;
  } else if (action === "remove") {
    const tabs = editableTabs.value;
    let activeName = editableTabsValue.value;
    if (activeName === targetName) {
      tabs.forEach((tab, index) => {
        if (tab.name === targetName) {
          const nextTab = tabs[index + 1] || tabs[index - 1];
          if (nextTab) {
            activeName = nextTab.name;
          }
        }
      });
    }
    editableTabsValue.value = activeName;
    editableTabs.value = tabs.filter((tab) => tab.name !== targetName);
  }
};
</script>
