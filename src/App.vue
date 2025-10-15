<template>
  <ModelViewer
    model-path="/file/information.glb"
    texture-path="/textures/img3.jpg"      
    normal-map-path="/textures/img2.jpg"          
    ao-map-path="/textures/img1.jpg"          
    metalness-map-path="/textures/img4.jpg" 
  />
</template>

<script setup>
import ModelViewer from "@/components/ModelViewer.vue";
</script>