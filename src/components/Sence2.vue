<template>
  <div id="three-datacenter"></div>
  <Popover
    ref="popoverRef"
    :top="popoverTop"
    :left="popoverLeft"
    :data="popoverData"
  ></Popover>
  
  <!-- 加载动画 -->
  <div class="loading-container" v-if="isLoading">
    <div class="loading-content">
      <div class="loading-spinner">
        <div class="spinner-circle"></div>
        <div class="spinner-line-mask">
          <div class="spinner-line"></div>
        </div>
        <div class="loading-percentage">{{ Math.floor(loadingPercentage) }}%</div>
      </div>
      <div class="loading-text">{{ loadingMessage }}</div>
      <div class="loading-progress-bar">
        <div class="progress-bar-fill" :style="{ width: loadingPercentage + '%' }"></div>
      </div>
      <div class="loading-details">正在加载机房模型资源...</div>
    </div>
    <div class="tech-decoration top-left"></div>
    <div class="tech-decoration top-right"></div>
    <div class="tech-decoration bottom-left"></div>
    <div class="tech-decoration bottom-right"></div>
  </div>
  
  <div class="back-button" @click="navigateBackToOffice">
    <span>返回办公楼</span>
  </div>
</template>

<script lang="ts" setup name="Sence2">
/* eslint-disable */
import { ref, onMounted, onBeforeUnmount, type Ref, computed } from 'vue';
import Viewer, { type Animate } from '@/modules/Viewer';
import ModelLoader from '@/modules/ModelLoder';
import * as THREE from 'three';
import gsap from 'gsap';
import Event from '@/modules/Viewer/Events';
import BoxHelperWrap from '@/modules/BoxHelperWrap';
import { checkNameIncludes, findParent } from '@/utils';

// Import Popover component
import Popover from './Popover/index.vue';

// 加载状态和进度
const isLoading = ref(true);
const loadingPercentage = ref(0);
const loadingStage = ref(0);
const loadingStages = [
  '初始化场景...',
  '加载模型资源...',
  '处理材质...',
  '优化渲染...',
  '准备完成...'
];
const loadingMessage = computed(() => {
  return loadingStages[loadingStage.value];
});

// 模型加载相关变量
const totalModelsToLoad = 2; // 总共需要加载的模型数量（datacenter和plane）
let loadedModels = 0;
const modelUrls = ['/models/datacenter.glb', '/models/plane.glb']; // 所有需要加载的模型URL
const modelProgress = new Map<string, { loaded: number, total: number }>(); // 跟踪每个模型的加载进度

let viewer: Viewer;
let modelLoader: ModelLoader;
let boxHelperWrap: BoxHelperWrap;
let dataCenter: any = null;
let oldDataCenter: any = null;

// Add popover related refs
const popoverRef: Ref = ref(null);
const popoverTop = ref(0);
const popoverLeft = ref(0);
const popoverData = ref<any>({});

onMounted(() => {
  init();
  
  // 初始化加载进度
  initLoadingProgress();
  
  loadDatacenterModel();
});

onBeforeUnmount(() => {
  // Clean up resources when component is unmounted
  if (viewer) {
    viewer.destroy();
  }
});

// 初始化加载进度
const initLoadingProgress = () => {
  // 初始进度为0
  loadingPercentage.value = 0;
  loadingStage.value = 0;
  
  // 阶段1: 初始化场景 (0-10%)
  gsap.to(loadingPercentage, {
    value: 10,
    duration: 0.5,
    ease: "power1.inOut",
    onComplete: () => {
      loadingStage.value = 1; // 进入"加载模型资源"阶段
    }
  });
};

// 更新加载进度
const updateLoadingProgress = () => {
  loadedModels++;
  
  // 如果所有模型都已加载，则隐藏加载动画
  if (loadedModels >= totalModelsToLoad) {
    loadingStage.value = 4; // 进入"准备完成"阶段
    
    // 确保进度到100%
    gsap.to(loadingPercentage, {
      value: 100,
      duration: 0.5,
      ease: "power1.out",
      onComplete: () => {
        // 然后淡出加载动画
        setTimeout(() => {
          isLoading.value = false;
        }, 800);
      }
    });
  }
};

// 处理模型加载进度
const handleModelProgress = (url: string, loaded: number, total: number) => {
  // 更新模型加载进度
  modelProgress.set(url, { loaded, total });
  
  // 计算所有模型的总加载进度
  let totalLoaded = 0;
  let totalSize = 0;
  
  modelProgress.forEach(progress => {
    totalLoaded += progress.loaded;
    totalSize += progress.total;
  });
  
  // 计算总体加载百分比 (10-90% 范围内)
  const baseProgress = 10; // 初始化场景已经完成的进度
  const maxProgress = 90; // 模型加载最多到90%，剩下的10%留给处理和优化
  
  // 只有当有总大小时才计算百分比
  if (totalSize > 0) {
    const modelLoadingPercentage = (totalLoaded / totalSize) * 100;
    const scaledPercentage = baseProgress + (modelLoadingPercentage * (maxProgress - baseProgress) / 100);
    
    // 更新加载进度
    gsap.to(loadingPercentage, {
      value: scaledPercentage,
      duration: 0.2,
      ease: "power1.out"
    });
    // 根据加载进度更新加载阶段
    if (modelLoadingPercentage > 50 && loadingStage.value < 2) {
      loadingStage.value = 2; // 进入"处理材质"阶段
    }
    
    if (modelLoadingPercentage > 80 && loadingStage.value < 3) {
      loadingStage.value = 3; // 进入"优化渲染"阶段
    }
  }
};

const init = () => {
  viewer = new Viewer('three-datacenter');
  viewer.initRaycaster();

  modelLoader = new ModelLoader(viewer);
  // 设置加载进度回调
  modelLoader.setProgressCallback(handleModelProgress);
  
  boxHelperWrap = new BoxHelperWrap(viewer);

  viewer.emitter.on(Event.click.raycaster, (list: THREE.Intersection[]) => {
    onMouseClick(list);
  });

  viewer.emitter.on(Event.mousemove.raycaster, (list: THREE.Intersection[]) => {
    onMouseMove(list);
  });
  
  // Listen for camera change events
  // viewer.emitter.on(Event.camera.change, (cameraInfo) => {
  //   onCameraChange(cameraInfo);
  // });
};
const onCameraChange = (cameraInfo: any) => {
  console.log('Camera changed:', cameraInfo);
  console.log('Position:', 
    `x: ${cameraInfo.position.x.toFixed(2)}, `,
    `y: ${cameraInfo.position.y.toFixed(2)}, `,
    `z: ${cameraInfo.position.z.toFixed(2)}`
  );
};

const loadDatacenterModel = () => {
  modelLoader.loadModelToScene('/models/datacenter.glb', baseModel => {
    baseModel.setScalc(0.2);
    const model = baseModel.gltf.scene;
    model.position.set(0, 0, 0);
    model.name = '机房';
    baseModel.openCastShadow();

    dataCenter = baseModel;
    oldDataCenter = model.clone();

    const rackList: any[] = [];
    model.traverse(item => {
      if (checkIsRack(item)) {
        rackList.push(item);
      }
    });

    viewer.setRaycasterObjects(rackList);
    
    // 更新加载进度
    updateLoadingProgress();
    
    // Center the camera on the datacenter model
    centerCameraOnModel(model);
  });

  modelLoader.loadModelToScene('/models/plane.glb', baseModel => {
    const model = baseModel.gltf.scene;
    model.scale.set(0.0001 * 3, 0.0001 * 3, 0.0001 * 3)
    model.position.set(0, 0, 0);
    model.name = 'plane';
    baseModel.openCastShadow();

    const texture = (baseModel.object.children[0] as any).material.map;
    // console.log(texture, 'texture-------');
    const fnOnj = planeAnimate(texture);
    viewer.addAnimate(fnOnj);
    
    // 更新加载进度
    updateLoadingProgress();
  });
};

const centerCameraOnModel = (model: THREE.Object3D) => {
  // Calculate bounding box to get center and size
  const boundingBox = new THREE.Box3().setFromObject(model);
  const center = new THREE.Vector3();
  boundingBox.getCenter(center);
  
  // Calculate size for appropriate camera distance
  const size = new THREE.Vector3();
  boundingBox.getSize(size);
  const maxDim = Math.max(size.x, size.y, size.z);
  const fov = viewer.camera.fov * (Math.PI / 180);
  const cameraDistance = Math.abs(maxDim / Math.sin(fov / 2) * 0.8); // Adjust the multiplier as needed
  
  // Set initial camera position
  viewer.camera.position.set(center.x, center.y + maxDim * 0.2, center.z + cameraDistance);
  viewer.camera.lookAt(center);
  viewer.controls.target.copy(center);
  
  // Animate camera to final position
  gsap.to(viewer.camera.position, {
    x: 5.5,
    y: 1,
    z: 0,
    duration: 2,
    ease: "power2.inOut",
  });
};

function checkIsRack(obj: any): boolean {
  return checkNameIncludes(obj, 'rack');
}

const onMouseClick = (intersects: THREE.Intersection[]) => {
  if (!intersects.length) return;
  
  const selectedObject = intersects[0].object;
  console.log('Selected object:', selectedObject);
  
  // Handle rack selection here if needed
};

// Add function to update rack info in popover
const updateRackInfo = (name: string) => {
  if (name) {
    popoverRef.value.setShow(true, { name });
    const event = viewer.mouseEvent as MouseEvent;
    popoverTop.value = event.y + 10;
    popoverLeft.value = event.x + 10;
  } else {
    popoverRef.value.setShow(false);
  }
};
const planeAnimate = (texture: any): Animate => {
    // console.log(texture, 'texture');
    texture.wrapS = THREE.RepeatWrapping;
    texture.wrapT = THREE.RepeatWrapping;
    const animateFn = {
      fun: () => {
        const count = texture.repeat.y;
        if (count <= 10) {
          texture.repeat.x += 0.01;
          texture.repeat.y += 0.02;
        } else {
          texture.repeat.x = 0;
          texture.repeat.y = 0;
        }
      },
      content: viewer,
    };
    return animateFn;
}

const onMouseMove = (intersects: THREE.Intersection[]) => {
  if (!intersects.length) {
    popoverRef.value.setShow(false);
    boxHelperWrap.setVisible(false);
    return;
  }
  
  const selectedObject = intersects[0].object;
  const rack = findParent(selectedObject, checkIsRack);
  
  if (rack) {
    boxHelperWrap.attach(rack);
    // Display rack info in popover
    updateRackInfo(rack.name);
  } else {
    popoverRef.value.setShow(false);
  }
};

// 清除场景和资源的函数
const clearScene = () => {
  // 隐藏Popover
  if (popoverRef.value) {
    popoverRef.value.setShow(false);
  }
  
  // 清除BoxHelper
  if (boxHelperWrap) {
    boxHelperWrap.setVisible(false);
  }
  
  // 清除数据中心模型资源
  if (dataCenter && dataCenter.object) {
    dataCenter.object.traverse((child: any) => {
      if (child.geometry) {
        child.geometry.dispose();
      }
      if (child.material) {
        if (Array.isArray(child.material)) {
          child.material.forEach((material: THREE.Material) => material.dispose());
        } else if (child.material) {
          child.material.dispose();
        }
      }
    });
    
    // 从场景中移除对象
    viewer.scene.remove(dataCenter.object);
  }
  
  // 清除平面模型资源
  viewer.scene.traverse((child: any) => {
    if (child.name === 'plane') {
      if (child.geometry) {
        child.geometry.dispose();
      }
      if (child.material) {
        if (Array.isArray(child.material)) {
          child.material.forEach((material: THREE.Material) => material.dispose());
        } else if (child.material) {
          child.material.dispose();
        }
      }
      viewer.scene.remove(child);
    }
  });
  
  // 清除内存引用
  dataCenter = null;
  oldDataCenter = null;
  
  console.log("数据中心场景已清除，资源已释放");
};

// 返回到办公楼场景的函数
const navigateBackToOffice = () => {
  // 清除当前场景资源
  clearScene();
  
  // 发出自定义事件，通知切换回Sence组件
  const event = new CustomEvent('navigate-to-office');
  window.dispatchEvent(event);
  console.log("已发出导航事件，返回办公楼场景");
};
</script>

<script lang="ts">
export default {
  name: 'Sence2'
};
</script>

<style scoped>
#three-datacenter {
  height: 100%;
  width: 100%;
}

.back-button {
  position: absolute;
  top: 20px;
  left: 20px;
  background-color: rgba(29, 78, 216, 0.8);
  color: white;
  padding: 10px 15px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background-color 0.3s;
}

.back-button:hover {
  background-color: rgba(30, 64, 175, 0.9);
}

/* 加载动画样式 */
.loading-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.9);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  overflow: hidden;
}

.loading-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  color: #00ffff;
  position: relative;
  z-index: 2;
}

.loading-spinner {
  position: relative;
  width: 150px;
  height: 150px;
  margin-bottom: 20px;
}

.spinner-circle {
  position: absolute;
  width: 100%;
  height: 100%;
  border: 4px solid rgba(0, 255, 255, 0.1);
  border-radius: 50%;
  box-sizing: border-box;
  box-shadow: 0 0 20px rgba(0, 255, 255, 0.2);
}

.spinner-line-mask {
  position: absolute;
  width: 100%;
  height: 100%;
  border-radius: 50%;
  overflow: hidden;
  transform-origin: center;
  mask-image: linear-gradient(0deg, rgba(0, 0, 0, 1) 50%, rgba(0, 0, 0, 0) 50%);
  -webkit-mask-image: linear-gradient(0deg, rgba(0, 0, 0, 1) 50%, rgba(0, 0, 0, 0) 50%);
  animation: rotate 2s linear infinite;
}

.spinner-line {
  position: absolute;
  width: 100%;
  height: 100%;
  border: 4px solid transparent;
  border-top-color: #00ffff;
  border-left-color: #00ffff;
  border-radius: 50%;
  box-sizing: border-box;
  box-shadow: 0 0 10px rgba(0, 255, 255, 0.7);
}

.loading-percentage {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-size: 24px;
  font-weight: bold;
  font-family: 'Orbitron', sans-serif;
  text-shadow: 0 0 10px rgba(0, 255, 255, 0.7);
}

.loading-text {
  font-size: 18px;
  margin-top: 10px;
  text-shadow: 0 0 10px rgba(0, 255, 255, 0.7);
  letter-spacing: 2px;
  animation: pulse 1.5s infinite;
  margin-bottom: 15px;
}

.loading-progress-bar {
  width: 300px;
  height: 6px;
  background-color: rgba(0, 255, 255, 0.1);
  border-radius: 3px;
  overflow: hidden;
  position: relative;
  margin-bottom: 15px;
  box-shadow: 0 0 10px rgba(0, 255, 255, 0.3);
}

.progress-bar-fill {
  height: 100%;
  background-color: #00ffff;
  border-radius: 3px;
  transition: width 0.3s ease;
  position: relative;
  box-shadow: 0 0 10px rgba(0, 255, 255, 0.7);
}

.progress-bar-fill::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.4), transparent);
  animation: shimmer 1.5s infinite;
}

.loading-details {
  font-size: 14px;
  color: rgba(0, 255, 255, 0.7);
  margin-top: 5px;
}

/* 科技感网格背景 */
.loading-container::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-image: 
    linear-gradient(rgba(0, 255, 255, 0.05) 1px, transparent 1px),
    linear-gradient(90deg, rgba(0, 255, 255, 0.05) 1px, transparent 1px);
  background-size: 20px 20px;
  z-index: -1;
  animation: gridMove 20s linear infinite;
}

/* 科技感装饰元素 */
.tech-decoration {
  position: absolute;
  width: 150px;
  height: 150px;
  border: 2px solid rgba(0, 255, 255, 0.2);
  z-index: 1;
}

.tech-decoration::before,
.tech-decoration::after {
  content: '';
  position: absolute;
  background-color: #00ffff;
}

.tech-decoration::before {
  width: 30px;
  height: 2px;
}

.tech-decoration::after {
  width: 2px;
  height: 30px;
}

.top-left {
  top: 50px;
  left: 50px;
  border-right: none;
  border-bottom: none;
}

.top-left::before,
.top-left::after {
  top: 0;
  left: 0;
}

.top-right {
  top: 50px;
  right: 50px;
  border-left: none;
  border-bottom: none;
}

.top-right::before,
.top-right::after {
  top: 0;
  right: 0;
}

.top-right::before {
  right: 0;
}

.bottom-left {
  bottom: 50px;
  left: 50px;
  border-right: none;
  border-top: none;
}

.bottom-left::before,
.bottom-left::after {
  bottom: 0;
  left: 0;
}

.bottom-right {
  bottom: 50px;
  right: 50px;
  border-left: none;
  border-top: none;
}

.bottom-right::before,
.bottom-right::after {
  bottom: 0;
  right: 0;
}

/* 动画 */
@keyframes rotate {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

@keyframes pulse {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}

@keyframes gridMove {
  0% {
    background-position: 0 0;
  }
  100% {
    background-position: 40px 40px;
  }
}

@keyframes shimmer {
  0% {
    transform: translateX(-100%);
  }
  100% {
    transform: translateX(100%);
  }
}
</style> 