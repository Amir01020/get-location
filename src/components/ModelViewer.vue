<template>
    <div class="model-viewer">
      <!-- 3D контейнер -->
      <div ref="containerRef" class="canvas-container"></div>
  
      <!-- Индикатор загрузки -->
      <div v-if="isLoading" class="loading">
        <div class="spinner"></div>
        <div>Загрузка модели... {{ loadingProgress }}%</div>
      </div>
  
      <!-- Ошибка -->
      <div v-if="error" class="error">
        <h3>⚠️ Ошибка загрузки</h3>
        <p>{{ error }}</p>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, onMounted, onBeforeUnmount } from 'vue';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
  
  // Props - МОДЕЛЬ + 4 ТЕКСТУРЫ
  const props = defineProps({
    modelPath: {
      type: String,
      required: true,
      default: '/file/weapon.glb'
    },
    texturePath: {
      type: String,
      required: true,
      default: null
    },
    normalMapPath: {
      type: String,
      required: true,
      default: null
    },
    aoMapPath: {
      type: String,
      required: true,
      default: null
    },
    metalnessMapPath: {
      type: String,
      required: true,
      default: null
    }
  });
  
  // Emits
  const emit = defineEmits(['loaded', 'error', 'progress']);
  
  // Refs
  const containerRef = ref(null);
  const isLoading = ref(true);
  const loadingProgress = ref(0);
  const error = ref(null);
  
  // Three.js объекты
  let scene, camera, renderer, controls, model, originalScale;
  let animationId = null;
  
  // Текстуры
  let diffuseTexture = null;
  let normalTexture = null;
  let aoTexture = null;
  let metalnessTexture = null;
  
  // ============================================
  // ИНИЦИАЛИЗАЦИЯ СЦЕНЫ
  // ============================================
  const initScene = () => {
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x2a2e35);
  
    const container = containerRef.value;
    camera = new THREE.PerspectiveCamera(
      40,
      container.clientWidth / container.clientHeight,
      0.1,
      1000
    );
    
    camera.position.set(0, 1.5, 4);
  
    renderer = new THREE.WebGLRenderer({ 
      antialias: true, 
      alpha: true
    });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.setSize(container.clientWidth, container.clientHeight);
    renderer.outputEncoding = THREE.sRGBEncoding;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.5; // Умеренная яркость
    
    renderer.shadowMap.enabled = false;
    
    container.appendChild(renderer.domElement);
  
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.minDistance = 2;
    controls.maxDistance = 8;
    controls.target.set(0, 0, 0);
    controls.enablePan = true;
  
    // ============================================
    // ☀️ СБАЛАНСИРОВАННОЕ ОСВЕЩЕНИЕ
    // ============================================
    
    const ambientLight = new THREE.AmbientLight(0xffffff, 1.0);
    scene.add(ambientLight);
  
    const sunLight = new THREE.DirectionalLight(0xffffff, 1.5);
    sunLight.position.set(-12, 3, 10);
    scene.add(sunLight);
  
    const fillLight = new THREE.DirectionalLight(0xffffff, 0.6);
    fillLight.position.set(12, 5, 8);
    scene.add(fillLight);
  
    const topLight = new THREE.DirectionalLight(0xffffff, 0.8);
    topLight.position.set(0, 20, 0);
    scene.add(topLight);
  
    const rimLight = new THREE.DirectionalLight(0xffffff, 0.5);
    rimLight.position.set(0, 3, -15);
    scene.add(rimLight);
  };
  
  // Загрузка всех текстур
  const loadTextures = async () => {
    const textureLoader = new THREE.TextureLoader();
    const promises = [];
    
    // 1. 🎨 DIFFUSE - Основная цветная текстура
    if (props.texturePath) {
      const diffusePromise = new Promise((resolve) => {
        textureLoader.load(
          props.texturePath,
          (texture) => {
            texture.encoding = THREE.sRGBEncoding;
            texture.flipY = false;
            texture.anisotropy = renderer.capabilities.getMaxAnisotropy();
            diffuseTexture = texture;
            console.log('✅ Diffuse текстура загружена');
            resolve();
          },
          undefined,
          (err) => {
            console.error('❌ Ошибка загрузки Diffuse:', err);
            resolve();
          }
        );
      });
      promises.push(diffusePromise);
    }
  
    // 2. 🔶 NORMAL MAP - Рельеф
    if (props.normalMapPath) {
      const normalPromise = new Promise((resolve) => {
        textureLoader.load(
          props.normalMapPath,
          (texture) => {
            texture.flipY = false;
            texture.anisotropy = renderer.capabilities.getMaxAnisotropy();
            normalTexture = texture;
            console.log('✅ Normal Map загружена');
            resolve();
          },
          undefined,
          (err) => {
            console.error('❌ Ошибка загрузки Normal Map:', err);
            resolve();
          }
        );
      });
      promises.push(normalPromise);
    }
  
    // 3. 🌑 AO MAP - Тени
    if (props.aoMapPath) {
      const aoPromise = new Promise((resolve) => {
        textureLoader.load(
          props.aoMapPath,
          (texture) => {
            texture.flipY = false;
            texture.anisotropy = renderer.capabilities.getMaxAnisotropy();
            aoTexture = texture;
            console.log('✅ AO Map загружена');
            resolve();
          },
          undefined,
          (err) => {
            console.error('❌ Ошибка загрузки AO Map:', err);
            resolve();
          }
        );
      });
      promises.push(aoPromise);
    }
  
    // 4. 💎 METALNESS MAP - Металличность
    if (props.metalnessMapPath) {
      const metalnessPromise = new Promise((resolve) => {
        textureLoader.load(
          props.metalnessMapPath,
          (texture) => {
            texture.flipY = false;
            texture.anisotropy = renderer.capabilities.getMaxAnisotropy();
            metalnessTexture = texture;
            console.log('✅ Metalness Map загружена');
            resolve();
          },
          undefined,
          (err) => {
            console.error('❌ Ошибка загрузки Metalness Map:', err);
            resolve();
          }
        );
      });
      promises.push(metalnessPromise);
    }
    
    await Promise.all(promises);
    console.log('✅ Все 4 текстуры загружены');
  };
  
  // Загрузка модели
  const loadModel = () => {
    const loader = new GLTFLoader();
  
    loader.load(
      props.modelPath,
      (gltf) => {
        model = gltf.scene;
  
        // Применяем ВСЕ текстуры к модели
        model.traverse((child) => {
          if (child.isMesh) {
            child.castShadow = false;
            child.receiveShadow = false;
            
            // ============================================
            // 🎨 ПОЛНЫЙ PBR МАТЕРИАЛ СО ВСЕМИ ТЕКСТУРАМИ
            // ============================================
            const newMaterial = new THREE.MeshStandardMaterial({
              // 1. Основной цвет (Diffuse/Albedo)
              map: diffuseTexture || null,
              
              // 2. Рельеф (Normal Map)
              normalMap: normalTexture || null,
              normalScale: new THREE.Vector2(1.0, 1.0), // Сила рельефа
              
              // 3. Ambient Occlusion (тени в углублениях)
              aoMap: aoTexture || null,
              aoMapIntensity: 1.5, // Сила теней
              
              // 4. Металличность
              metalnessMap: metalnessTexture || null,
              metalness: 0.8, // Базовое значение металличности
              
              // Дополнительные настройки
              color: 0xffffff,
              roughness: 0.4, // Шероховатость (можно добавить roughnessMap)
              envMapIntensity: 0.3,
              
              // Roughness Map отсутствует, используем базовое значение
              roughnessMap: null
            });
  
            // Encoding для цветной текстуры
            if (newMaterial.map) {
              newMaterial.map.encoding = THREE.sRGBEncoding;
            }
            
            // Для AO нужен второй UV канал
            if (aoTexture && child.geometry.attributes.uv) {
              child.geometry.setAttribute('uv2', child.geometry.attributes.uv);
            }
            
            child.material = newMaterial;
            child.material.needsUpdate = true;
            
            console.log('✅ Материал применён к:', child.name || 'mesh');
          }
        });
  
        // Автоцентрование
        const box = new THREE.Box3().setFromObject(model);
        const center = box.getCenter(new THREE.Vector3());
        const size = box.getSize(new THREE.Vector3());
  
        model.position.sub(center);
  
        const maxDim = Math.max(size.x, size.y, size.z);
        originalScale = 2 / maxDim;
        model.scale.setScalar(originalScale);
  
        scene.add(model);
  
        controls.target.set(0, 0, 0);
        controls.update();
  
        isLoading.value = false;
        emit('loaded', model);
        console.log('✅ Модель загружена со всеми текстурами!');
      },
      (progress) => {
        if (progress.total > 0) {
          loadingProgress.value = Math.round((progress.loaded / progress.total) * 100);
          emit('progress', loadingProgress.value);
        }
      },
      (err) => {
        error.value = `Не удалось загрузить модель: ${err.message}`;
        isLoading.value = false;
        emit('error', err);
        console.error('❌ Ошибка загрузки модели:', err);
      }
    );
  };
  
  // Анимация
  const animate = () => {
    animationId = requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
  };
  
  // Обработка resize
  const handleResize = () => {
    if (!containerRef.value || !camera || !renderer) return;
    const container = containerRef.value;
    camera.aspect = container.clientWidth / container.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(container.clientWidth, container.clientHeight);
  };
  
  // Lifecycle hooks
  onMounted(async () => {
    initScene();
    await loadTextures();
    loadModel();
    animate();
    window.addEventListener('resize', handleResize);
  });
  
  onBeforeUnmount(() => {
    if (animationId) cancelAnimationFrame(animationId);
    window.removeEventListener('resize', handleResize);
    
    if (renderer) {
      renderer.dispose();
      if (containerRef.value?.contains(renderer.domElement)) {
        containerRef.value.removeChild(renderer.domElement);
      }
    }
    if (controls) controls.dispose();
    
    // Очистка текстур
    if (diffuseTexture) diffuseTexture.dispose();
    if (normalTexture) normalTexture.dispose();
    if (aoTexture) aoTexture.dispose();
    if (metalnessTexture) metalnessTexture.dispose();
  });
  </script>
  
  <style scoped>
  .model-viewer {
    position: relative;
    width: 100%;
    height: 100vh;
    background: #2a2e35;
  }
  
  .canvas-container {
    width: 100%;
    height: 100%;
    background: #2a2e35;
  }
  
  .loading {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    color: #fff;
    font-size: 18px;
    text-align: center;
    z-index: 100;
  }
  
  .spinner {
    border: 3px solid rgba(255, 255, 255, 0.1);
    border-top: 3px solid #667eea;
    border-radius: 50%;
    width: 40px;
    height: 40px;
    animation: spin 1s linear infinite;
    margin: 0 auto 15px;
  }
  
  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
  
  .error {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background: rgba(220, 38, 38, 0.9);
    color: white;
    padding: 20px 30px;
    border-radius: 12px;
    max-width: 500px;
    text-align: center;
    z-index: 100;
  }
  </style>