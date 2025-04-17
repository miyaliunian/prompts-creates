<script lang="ts">
import { defineComponent } from "vue";
import ImageUploader from "./components/ImageUploader.vue";

export default defineComponent({
  name: "App",
  components: {
    ImageUploader,
  },
});
</script>

<template>
  <a-layout class="layout">
    <a-layout-content class="content">
      <ImageUploader />
    </a-layout-content>
  </a-layout>
</template>

<style>
.layout {
  height: 100%;
  width: 100%;
}

.header {
  display: flex;
  align-items: center;
  background: #fff;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  z-index: 1;
}

.logo {
  color: #1890ff;
  font-size: 18px;
  font-weight: bold;
}

.content {
  background: #f5f5f5;
  display: flex;
  justify-content: center;
}

.content > * {
  width: 100%;
}

.footer {
  text-align: center;
  color: rgba(0, 0, 0, 0.45);
}
</style>
