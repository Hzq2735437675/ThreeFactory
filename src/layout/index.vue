<template>
  <div class="layout">
    <Header>数据中心可视化系统</Header>
    <div class="layout-content">
      <!-- <div class="layout-content-left">
        <Panel>
          <div>向西28° 2023/04/02 12:00 正常</div>
          <div>向西28° 2023/04/02 12:00 正常</div>
          <div>向西28° 2023/04/02 12:00 正常</div>
        </Panel>
        <Panel>
          <PieChart></PieChart>
        </Panel>
      </div>
      <div class="layout-content-right">
        <Panel>
          <LineCharts></LineCharts>
        </Panel>
      </div> -->
      <Sence v-if="currentView === 'sence'"></Sence>
      <Sence2 v-if="currentView === 'sence2'"></Sence2>
    </div>
  </div>
</template>

<script setup lang="ts" name="Layout">
import { ref, onMounted, onBeforeUnmount } from 'vue';
import Header from './Header.vue';
import Sence from '@/components/Sence.vue';
import Sence2 from '@/components/Sence2.vue';
import Panel from '@/components/Panel/index.vue';
// import LineCharts from '@/components/Charts/LineChart.vue';
// import PieChart from '@/components/Charts/PieChart.vue';

// 当前显示的视图组件
const currentView = ref('sence');

// 监听自定义事件，切换到数据中心视图
const handleNavigateToDatacenter = () => {
  currentView.value = 'sence2';
  console.log('已切换到数据中心视图');
};

// 监听自定义事件，返回到办公楼视图
const handleNavigateToOffice = () => {
  currentView.value = 'sence';
  console.log('已返回到办公楼视图');
};

onMounted(() => {
  // 添加事件监听器
  window.addEventListener('navigate-to-datacenter', handleNavigateToDatacenter);
  window.addEventListener('navigate-to-office', handleNavigateToOffice);
});

onBeforeUnmount(() => {
  // 移除事件监听器，防止内存泄漏
  window.removeEventListener('navigate-to-datacenter', handleNavigateToDatacenter);
  window.removeEventListener('navigate-to-office', handleNavigateToOffice);
});
</script>

<script lang="ts">
export default {
  name: 'Layout'
};
</script>

<style scope>
.layout {
  width: 100vw;
  height: 100vh;
}
.layout-content {
  width: 100vw;
  height: 100vh;
  background-color: #fff;
}
.layout-content-left {
  position: absolute;
  left: 0.3rem;
  top: 70px;
  width: 30vw;
}
.layout-content-right {
  position: absolute;
  right: 0.6rem;
  top: 70px;
  width: 30vw;
}
.panel {
  margin-bottom: 0.3rem;
}
</style>
