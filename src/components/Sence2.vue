<template>
  <div id="three-datacenter"></div>
  <Popover
    ref="popoverRef"
    :top="popoverTop"
    :left="popoverLeft"
    :data="popoverData"
  ></Popover>
  <div class="back-button" @click="navigateBackToOffice">
    <span>返回办公楼</span>
  </div>
</template>

<script lang="ts" setup name="Sence2">
/* eslint-disable */
import { ref, onMounted, onBeforeUnmount, type Ref } from 'vue';
import Viewer, { type Animate } from '@/modules/Viewer';
import ModelLoader from '@/modules/ModelLoder';
import * as THREE from 'three';
import gsap from 'gsap';
import Event from '@/modules/Viewer/Events';
import BoxHelperWrap from '@/modules/BoxHelperWrap';
import { checkNameIncludes, findParent } from '@/utils';

// Import Popover component
import Popover from './Popover/index.vue';

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
  loadDatacenterModel();
});

onBeforeUnmount(() => {
  // Clean up resources when component is unmounted
  if (viewer) {
    viewer.destroy();
  }
});

const init = () => {
  viewer = new Viewer('three-datacenter');
  viewer.initRaycaster();

  modelLoader = new ModelLoader(viewer);
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
</style> 