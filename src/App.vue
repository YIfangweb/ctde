<script setup>
import AMapContainer from './components/AMapContainer.vue'
</script>

<template>
  <AMapContainer />
</template>

<style>
html, body {
  margin: 0;
  padding: 0;
  height: 100%;
  width: 100%;
  font-family: Arial, sans-serif;
  overflow: hidden;
}

/* 确保#app占满整个屏幕 */
#app {
  height: 100vh;
  width: 100vw;
}
</style>
