<template>


  <ModelViewer 
    modelPath="/file/information.glb"
    texturePath="/textures/img3.jpg"
    normalMapPath="/textures/img1.png"
    aoMapPath="/textures/img2.jpg"
    metalnessMapPath="/textures/img4.jpg"
  />
</template>

<script setup>
import ModelViewer from "./components/ModelViewer.vue";
</script>

<style scoped>
.app-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

h1 {
  text-align: center;
  color: #374151;
  margin-bottom: 20px;
}

.test-controls {
  margin-top: 20px;
  display: flex;
  gap: 12px;
  justify-content: center;
}

.test-btn {
  padding: 10px 20px;
  background: #3390ec;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s;
}

.test-btn:hover {
  background: #2b7cd3;
  transform: translateY(-1px);
}
</style>