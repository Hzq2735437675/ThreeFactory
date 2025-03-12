<template>
  <div id="three"></div>
  <Popover
    ref="popoverRef"
    :top="popoverTop"
    :left="popoverLeft"
    :data="popoverData"
  ></Popover>
  <div class="control-buttons">
    <!-- <button class="control-btn" @click="explodeModel">模型分解</button> -->
    <!-- <button class="control-btn" @click="increaseExplode" v-if="isExploded">增加分解</button> -->
    <!-- <button class="control-btn" @click="resetModel">还原模型</button> -->
    <button class="control-btn" @click="toggleGlowEffect">{{ isGlowing ? '关闭发光' : '开启发光' }}</button>
  </div>
</template>

<script lang="ts" setup name="Sence">
/* eslint-disable */
import { ref, onMounted, type Ref } from 'vue';
import Viewer, { type Animate } from '@/modules/Viewer';
import Floors from '@/modules/Floors';
import ModelLoader from '@/modules/ModelLoder';
import * as THREE from 'three';
import gsap from 'gsap';
import Event from '@/modules/Viewer/Events';
import BoxHelperWrap from '@/modules/BoxHelperWrap';
import { checkNameIncludes, findParent } from '@/utils';

// Import the component directly
import Popover from './Popover/index.vue';

let viewer: Viewer;
let modelLoader: ModelLoader;
let boxHelperWrap: BoxHelperWrap;

const popoverRef: Ref = ref(null);
const popoverTop = ref(0);
const popoverLeft = ref(0);
const popoverData = ref<any>({});

let office: any = null;
let oldOffice: any = null;
let dataCenter: any = null;
let oldDataCenter: any = null;
let modelSelect = ['zuo0', 'zuo1', 'zuo2', 'zuo3', 'zuo4', 'zuo5'];
let modelSelectName = '';
let modelMoveName = '';
let isModelSelectName = false;

// 爆炸效果相关变量
let isExploded = false;
let originalPositions: Map<string, THREE.Vector3> = new Map();
let glowEffect: any = null;
let isGlowing = ref(false);
let explodeScalar = 0; // 爆炸系数，0表示未爆炸，1表示完全爆炸

onMounted(() => {
  init();
  initModel();

  viewer.scene.traverse((item: THREE.Object3D) => {
    // console.log(item, '0000000000');
  });
});

const init = () => {
  viewer = new Viewer('three');
  // viewer.addAxis();
  // viewer.addStats();
  viewer.initRaycaster();

  modelLoader = new ModelLoader(viewer);
  // const floors = new Floors(viewer);
  // floors.addGird();

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

// Function to handle camera change events
const onCameraChange = (cameraInfo: any) => {
  console.log('Camera changed:', cameraInfo);
  console.log('Position:', 
    `x: ${cameraInfo.position.x.toFixed(2)}, `,
    `y: ${cameraInfo.position.y.toFixed(2)}, `,
    `z: ${cameraInfo.position.z.toFixed(2)}`
  );
};

const initModel = () => {
  modelLoader.loadModelToScene('/models/zuo.glb', baseModel => {
    // console.log(baseModel, '1111111');

    baseModel.setScalc(0.01);
    const model = baseModel.gltf.scene;
    office = baseModel;
    office.object.rotation.y = Math.PI;
    office.object.position.set(0, 0, 0);
    // model.position.set(80, 2, 90);
    office.object.children.forEach((item: any) => {
      item.name = item.name.replace('zuo', '');
      if (item.name === 'ding') {
        item.name = 6;
      }
      item.name--;
    });
    office.object.children.sort((a: { name: number; }, b: { name: number; }) => a.name - b.name).forEach((v: { name: string; }) => {
      v.name = 'zuo' + v.name;
    });

    model.name = '办公楼';
    baseModel.openCastShadow();
    oldOffice = model.clone();

    const list: THREE.Object3D<THREE.Event>[] = [];
    model.traverse(item => {
      list.push(item);
    });
    viewer.setRaycasterObjects(list);
    
    // Add camera transition after model is loaded
    transitionCameraToNewPosition();
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


  // modelLoader.loadModelToScene('/models/datacenter.glb', baseModel => {
  //   // console.log(baseModel, '1111111');
  //   baseModel.setScalc(0.2);
  //   // baseModel.object.rotation.y = Math.PI / 2;
  //   const model = baseModel.gltf.scene;
  //   model.position.set(0, 0, 0);
  //   model.name = '机房';
  //   baseModel.openCastShadow();

  //   dataCenter = baseModel;
  //   oldDataCenter = model.clone();

  //   const rackList: any[] = [];
  //   model.traverse(item => {
  //     if (checkIsRack(item)) {
  //       rackList.push(item);
  //     }
  //   });
  //   // console.log(rackList, 'rackList------');

  //   viewer.setRaycasterObjects(rackList);

  // });
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

const onMouseClick = (intersects: THREE.Intersection[]) => {
  // 如果发光效果开启，只允许点击发光效果按钮，不处理其他点击事件
  if (isGlowing.value) {
    return;
  }
  
  if (!intersects.length) {
    if (isModelSelectName) {
      resetView();
    }
    return;
  }
  
  const selectedObject = intersects[0].object;

  let selectedObjectName = '';
  const findClickModel = (object: any) => {
    if (object.type === 'Group') {
      selectedObjectName = object.name;
    }
    if (object.parent && object.type !== 'Scene') {
      findClickModel(object.parent);
    }
  };
  findClickModel(selectedObject);

  // 点击楼房
  if (selectedObject.name.includes('zuo')) {
    selectOffice(selectedObject.parent);
  }
  // 点击其他区域
  else if (isModelSelectName) {
    resetView();
  }
};

function checkIsRack (obj: any): boolean {
  return checkNameIncludes(obj, 'rack');
}

const onMouseMove = (intersects: THREE.Intersection[]) => {
  // 如果发光效果开启，不执行悬停效果
  if (isGlowing.value) {
    return;
  }
  
  if (!intersects.length) {
    popoverRef.value.setShow(false);
    boxHelperWrap.setVisible(false);
    
    // 如果鼠标离开了楼层，并且不是在"楼层选中"模式下，恢复原始材质
    if (!isModelSelectName && modelMoveName) {
      // 恢复上一个悬停的楼层材质
      resetHoverEffect();
    }
    return;
  }
  const selectedObject = intersects[0].object || {};

  let selectedObjectName = '';
  const findClickModel = (object: any) => {
    if (object.name.includes('rack')) {
      selectedObjectName = object.name;
      return;
    }
    if (object.parent) {
      findClickModel(object.parent);
    }
  };
  findClickModel(selectedObject);

  const rack = findParent(selectedObject, checkIsRack);
  if (rack) {
    boxHelperWrap.attach(rack);
    updateRackInfo(rack.name);
  }

  // 处理楼层的鼠标悬停效果
  if (selectedObject.parent?.name && selectedObject.parent.name.includes('zuo')) {
    const floorName = selectedObject.parent.name;
    
    // 如果已经选中了某个楼层（处于"楼层选中"模式）
    if (isModelSelectName) {
      // 如果鼠标悬停在非选中的楼层上，不做任何处理
      // 这样可以防止透明楼层在鼠标悬停时变成黄色
      return;
    }
    
    // 记录当前悬停的楼层名称
    const previousHoverFloor = modelMoveName;
    modelMoveName = floorName;
    
    // 只有在悬停的楼层不是已选中的楼层时才应用悬停效果
    if (modelSelectName !== modelMoveName) {
      // 为当前悬停的楼层应用黄色高亮效果
      office.object.getObjectByName(floorName).traverse(function (child: any) {
        if (child.isMesh) {
          // 存储原始材质，以便鼠标离开时恢复
          if (!child.userData) child.userData = {};
          if (!child.userData.originalMaterial) {
            child.userData.originalMaterial = child.material;
          }
          
          child.material = new THREE.MeshPhongMaterial({
            side: THREE.DoubleSide,
            transparent: true,
            depthTest: false,
            depthWrite: true,
            color: 'yellow',
            opacity: 0.3,
          });
        }
      });
      
      // 如果之前悬停在其他楼层上，恢复那个楼层的材质
      if (previousHoverFloor && previousHoverFloor !== floorName) {
        resetFloorMaterial(previousHoverFloor);
      }
    }
  } else {
    // 鼠标不在任何楼层上，如果之前有悬停的楼层且不是在"楼层选中"模式下，恢复那个楼层的材质
    if (!isModelSelectName && modelMoveName) {
      resetHoverEffect();
    }
  }
};

// 重置悬停效果的函数
const resetHoverEffect = () => {
  if (modelMoveName) {
    resetFloorMaterial(modelMoveName);
    modelMoveName = '';
  }
};

// 重置楼层材质的函数
const resetFloorMaterial = (floorName: string) => {
  const floor = office.object.getObjectByName(floorName);
  if (!floor) return;
  
  // 如果在"楼层选中"模式下，且这不是被选中的楼层，恢复为透明材质
  if (isModelSelectName && floorName !== modelSelectName) {
    floor.traverse(function (child: any) {
      if (child.isMesh) {
        // 创建透明材质
        const transparentMaterial = new THREE.MeshBasicMaterial();
        // 如果有原始材质，克隆它的一些属性
        if (child.userData && child.userData.originalMaterial) {
          transparentMaterial.copy(child.userData.originalMaterial);
        }
        transparentMaterial.transparent = true;
        transparentMaterial.opacity = 0.2; // 非常透明
        child.material = transparentMaterial;
      }
    });
  } 
  // 否则恢复为原始材质
  else {
    let oldmodel = oldOffice.getObjectByName(floorName);
    floor.traverse(function (child: any) {
      if (child.isMesh) {
        const originalMesh = oldmodel.getObjectByName(child.name);
        if (originalMesh && originalMesh.material) {
          child.material = originalMesh.material;
        }
      }
    });
  }
};

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

const selectOffice = (model: any) => {
  modelSelectName = model.name;
  let oldmodel = oldOffice.getObjectByName(modelSelectName);
  let modelSelectIndex = modelSelect.findIndex(v => v === modelSelectName);
  
  // Reset all floors to visible first
  office.object.children.forEach((child: any) => {
    child.visible = true;
    // Reset any previous transformations
    if (child.userData.position) {
      gsap.to(child.position, {
        y: oldOffice.getObjectByName(child.name).position.y,
        duration: 1,
        ease: "power1.inOut",
        onComplete: () => {
          child.userData.position = false;
        },
      });
    }
    
    // Reset materials to original for all floors
    let floorOldModel = oldOffice.getObjectByName(child.name);
    child.traverse(function (mesh: { isMesh: any; material: any; name: any; }) {
      if (mesh.isMesh && floorOldModel) {
        const originalMesh = floorOldModel.getObjectByName(mesh.name);
        if (originalMesh && originalMesh.material) {
          mesh.material = originalMesh.material;
        }
      }
    });
  });
  
  // After a short delay, make other floors transparent instead of hiding them
  setTimeout(() => {
    // Make other floors transparent instead of hiding them
    office.object.children.forEach((child: any) => {
      if (child.name !== modelSelectName) {
        // Apply transparent material to other floors
        child.traverse(function (mesh: { isMesh: any; material: THREE.Material; }) {
          if (mesh.isMesh) {
            // Clone the material to avoid affecting other instances
            const transparentMaterial = mesh.material.clone();
            transparentMaterial.transparent = true;
            transparentMaterial.opacity = 0.2; // Very transparent
            mesh.material = transparentMaterial;
          }
        });
      }
    });
    
    // Get the selected floor for camera positioning
    const selectedFloor = office.object.getObjectByName(modelSelectName);
    
    // Calculate target camera position to face the selected floor
    // Get the center position of the selected floor
    const boundingBox = new THREE.Box3().setFromObject(selectedFloor);
    const center = new THREE.Vector3();
    boundingBox.getCenter(center);
    
    // Calculate appropriate camera distance based on bounding box size
    const size = new THREE.Vector3();
    boundingBox.getSize(size);
    const maxDim = Math.max(size.x, size.y, size.z);
    const fov = viewer.camera.fov * (Math.PI / 180);
    const cameraDistance = Math.abs(maxDim / Math.sin(fov / 2) * 0.5);
    
    // 重置控制器目标点到楼层中心，确保相机旋转正确
    gsap.to(viewer.controls.target, {
      x: center.x,
      y: center.y,
      z: center.z,
      duration: 1,
      ease: "power2.inOut",
      onUpdate: () => {
        viewer.controls.update();
      }
    });
    
    // 第一阶段：相机过渡到对应楼层
    // 使用相对于楼层中心的位置，而不是固定坐标
    const firstStagePosition = new THREE.Vector3(
      center.x,
      center.y,
      center.z - 4 // 在楼层前方4个单位
    );
    
    gsap.to(viewer.camera.position, {
      x: firstStagePosition.x,
      y: firstStagePosition.y,
      z: firstStagePosition.z,
      duration: 2,
      ease: "power2.inOut",
      onUpdate: () => {
        viewer.camera.lookAt(center);
      },
      onComplete: () => {
        viewer.controls.update();
        
        // 第二阶段：相机继续靠近楼层
        // 计算更近的位置，相对于楼层中心
        const secondStagePosition = new THREE.Vector3(
          center.x,
          center.y,
          center.z - 2 // 在楼层前方2个单位，更近了
        );
        
        gsap.to(viewer.camera.position, {
          x: secondStagePosition.x,
          y: secondStagePosition.y,
          z: secondStagePosition.z,
          duration: 1,
          ease: "power1.in",
          onUpdate: () => {
            viewer.camera.lookAt(center);
          },
          onComplete: () => {
            viewer.controls.update();
            // 清空场景资源
            clearScene();
            
            // 切换到Sence2组件
            navigateToDatacenterScene();
          }
        });
      }
    });
    
    // Animate the camera's field of view to "zoom in"
    gsap.to(viewer.camera, {
      fov: 20, // A smaller FOV gives a zoom-in effect
      duration: 2,
      ease: "power2.inOut",
      onUpdate: () => {
        viewer.camera.updateProjectionMatrix();
      }
    });
    
  }, 300); // Short delay before applying transparency
  
  // Set flag to indicate a floor is selected
  isModelSelectName = true;
};

// Function to clear the scene and dispose of resources
const clearScene = () => {
  // Dispose of geometries and materials
  office.object.traverse((child: any) => {
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
  
  // Remove objects from scene
  viewer.scene.remove(office.object);
  
  // Clear memory references
  office = null;
  oldOffice = null;
  
  console.log("Scene cleared and resources disposed");
};

// 切换到数据中心场景的函数
const navigateToDatacenterScene = () => {
  // 发出自定义事件，App.vue 监听此事件来切换组件
  const event = new CustomEvent('navigate-to-datacenter');
  window.dispatchEvent(event);
  console.log("已发出导航事件，显示数据中心场景");
};

// Add a function to reset the view when clicking elsewhere
const resetView = () => {
  // Make all floors visible again
  office.object.children.forEach((child: any) => {
    child.visible = true;
    
    // Reset materials to original
    let oldmodel = oldOffice.getObjectByName(child.name);
    child.traverse(function (mesh: { isMesh: any; material: any; name: any; }) {
      if (mesh.isMesh) {
        mesh.material = oldmodel.getObjectByName(mesh.name).material;
      }
    });
  });
  
  // Reset camera position and FOV
  gsap.to(viewer.camera.position, {
    x: 0,
    y: 1.85,
    z: -5,
    duration: 2,
    ease: "power2.inOut",
  });
  
  gsap.to(viewer.camera, {
    fov: 25, // Reset to original FOV
    duration: 2,
    ease: "power2.inOut",
    onUpdate: () => {
      viewer.camera.updateProjectionMatrix();
    }
  });
  
  // Reset controls target
  gsap.to(viewer.controls.target, {
    x: 0,
    y: 0,
    z: 0,
    duration: 2,
    ease: "power2.inOut",
    onUpdate: () => {
      viewer.controls.update();
    }
  });
  
  // Reset flag
  isModelSelectName = false;
  modelSelectName = '';
};

// Add new function for camera transition
const transitionCameraToNewPosition = () => {
  // Store initial camera position
  const initialPosition = {
    x: viewer.camera.position.x,
    y: viewer.camera.position.y,
    z: viewer.camera.position.z
  };
  
  // Target position
  const targetPosition = {
    x: 0,
    y: 1.85,
    z: -5
  };
  
  // Create GSAP animation for smooth transition
  gsap.to(initialPosition, {
    x: targetPosition.x,
    y: targetPosition.y,
    z: targetPosition.z,
    duration: 2.5, // Animation duration in seconds
    ease: "power2.inOut", // Easing function for smooth transition
    onUpdate: () => {
      // Update camera position during animation
      viewer.camera.position.set(initialPosition.x, initialPosition.y, initialPosition.z);
      viewer.camera.lookAt(0, 0, 0); // Keep camera looking at the center
    },
    onComplete: () => {
      console.log("Camera transition completed");
      // Ensure final position is set exactly
      viewer.camera.position.set(targetPosition.x, targetPosition.y, targetPosition.z);
      viewer.camera.lookAt(0, 0, 0);
    }
  });
};

// 获取物体的世界中心位置
function getWorldCenterPosition(box: THREE.Box3, scalar = 0.5): THREE.Vector3 {
  // 计算包围盒的中心点，scalar参数可以调整中心点的位置
  return new THREE.Vector3().addVectors(box.max, box.min).multiplyScalar(scalar);
}

// 初始化爆炸数据保存到每个mesh的userData上
function initExplodeData(modelObject: THREE.Object3D) {
  if (!modelObject) return;

  // 计算整个模型的包围盒和中心点
  const explodeBox = new THREE.Box3();
  explodeBox.setFromObject(modelObject);
  const explodeCenter = getWorldCenterPosition(explodeBox);

  const meshBox = new THREE.Box3();

  // 遍历整个模型，保存数据到userData上，以便爆炸函数使用
  modelObject.traverse(function (value: any) {
    // 跳过标记、线条和精灵等非网格对象
    if (value.isMark || value.isMarkChild || value.isLine || value.isSprite) return;
    if (value.isMesh) {
      // 计算每个网格的包围盒和中心点
      meshBox.setFromObject(value);
      const meshCenter = getWorldCenterPosition(meshBox);
      
      // 爆炸方向：从模型中心指向网格中心的单位向量
      value.userData.worldDir = new THREE.Vector3()
        .subVectors(meshCenter, explodeCenter)
        .normalize();
      
      // 爆炸距离：网格中心点到爆炸中心点的距离向量
      value.userData.worldDistance = new THREE.Vector3().subVectors(meshCenter, explodeCenter);
      
      // 保存原始世界坐标，用于还原
      value.userData.originPosition = value.getWorldPosition(new THREE.Vector3());
      
      // 保存原始局部坐标，用于还原
      value.userData.originalLocalPosition = value.position.clone();
      
      // 保存网格中心点和爆炸中心点，用于计算爆炸后的位置
      value.userData.meshCenter = meshCenter.clone();
      value.userData.explodeCenter = explodeCenter.clone();
    }
  });
}

// 执行爆炸效果
function applyExplodeEffect(model: THREE.Object3D, scalar: number) {
  model.traverse(function (value: any) {
    // 跳过没有原始位置数据的网格
    if (!value.isMesh || !value.userData.originPosition) return;
    
    // 如果scalar为0，直接还原到原始局部坐标
    if (scalar === 0 && value.userData.originalLocalPosition) {
      value.position.copy(value.userData.originalLocalPosition);
      return;
    }
    
    // 计算爆炸距离：方向向量 * 距离 * 爆炸系数
    const distance = value.userData.worldDir
      .clone()
      .multiplyScalar(value.userData.worldDistance.length() * scalar);
    
    // 计算网格中心点和原始位置的偏移量
    const offset = new THREE.Vector3().subVectors(
      value.userData.meshCenter,
      value.userData.originPosition
    );
    
    // 计算爆炸后的新位置：爆炸中心 + 爆炸距离 - 偏移量
    const center = value.userData.explodeCenter;
    const newPos = new THREE.Vector3().copy(center).add(distance).sub(offset);
    
    // 将世界坐标转换为局部坐标，并应用到网格上
    const localPosition = value.parent?.worldToLocal(newPos.clone());
    localPosition && value.position.copy(localPosition);
  });
}

// 爆炸效果函数
const explodeModel = () => {
  if (!office) return;
  
  console.log("执行模型分解，当前状态:", { isExploded, explodeScalar });
  
  // 如果已经爆炸，则增加爆炸系数
  if (isExploded) {
    increaseExplode();
    return;
  }
  
  // 初始化爆炸数据（如果尚未初始化）
  if (!office.object.userData.explodeInitialized) {
    console.log("初始化爆炸数据");
    initExplodeData(office.object);
    office.object.userData.explodeInitialized = true;
    
    // 确保初始状态是scalar=0的状态
    applyExplodeEffect(office.object, 0);
  }
  
  // 使用GSAP动画过渡爆炸效果
  gsap.to({ value: 0 }, {
    value: 1,
    duration: 1.5,
    ease: "power2.out",
    onUpdate: function() {
      explodeScalar = this.targets()[0].value;
      applyExplodeEffect(office.object, explodeScalar);
    },
    onComplete: function() {
      isExploded = true;
      console.log("分解完成，当前状态:", { isExploded, explodeScalar });
    }
  });
};

// 增加分解程度函数
const increaseExplode = () => {
  if (!office || !isExploded) return;
  
  // 增加爆炸系数，最大为3
  const oldScalar = explodeScalar;
  explodeScalar = Math.min(explodeScalar + 0.5, 3);
  
  console.log("增加分解程度:", { oldScalar, newScalar: explodeScalar });
  
  // 应用新的爆炸效果
  applyExplodeEffect(office.object, explodeScalar);
};

// 还原模型函数
const resetModel = () => {
  if (!office) return;
  
  console.log("还原模型，当前状态:", { isExploded, explodeScalar });
  
  // 使用GSAP动画过渡回原始状态
  gsap.to({ value: explodeScalar }, {
    value: 0,
    duration: 1.5,
    ease: "power2.inOut",
    onUpdate: function() {
      explodeScalar = this.targets()[0].value;
      applyExplodeEffect(office.object, explodeScalar);
    },
    onComplete: function() {
      isExploded = false;
      
      // 确保完全还原到原始位置
      office.object.traverse((child: any) => {
        if (child.isMesh && child.userData && child.userData.originalLocalPosition) {
          // 使用保存的原始局部坐标
          child.position.copy(child.userData.originalLocalPosition);
        }
      });
      
      console.log("还原完成，当前状态:", { isExploded, explodeScalar });
    }
  });
  
  // 还原旋转
  office.object.traverse((child: any) => {
    if (child.isMesh) {
      gsap.to(child.rotation, {
        x: 0,
        y: 0,
        z: 0,
        duration: 1.5,
        ease: "power2.inOut"
      });
    }
  });
};

// 发光效果函数
const toggleGlowEffect = () => {
  if (!office) return;
  
  isGlowing.value = !isGlowing.value;
  
  if (isGlowing.value) {
    // 启用发光效果
    applyGlowEffect();
    
    // 如果当前有悬停效果，清除它
    if (modelMoveName) {
      resetHoverEffect();
    }
    
    console.log("发光效果已开启，悬停效果已禁用");
  } else {
    // 移除发光效果
    removeGlowEffect();
    console.log("发光效果已关闭，悬停效果已启用");
  }
};

// 应用发光效果
const applyGlowEffect = () => {
  if (!office) return;
  
  // 创建发光效果的着色器材质
  const vertexShader = `
    varying vec2 vUv;
    varying vec3 vPosition;
    varying vec3 vNormal;
    uniform float time;
    
    // 噪声函数
    vec3 mod289(vec3 x) { return x - floor(x * (1.0 / 289.0)) * 289.0; }
    vec4 mod289(vec4 x) { return x - floor(x * (1.0 / 289.0)) * 289.0; }
    vec4 permute(vec4 x) { return mod289(((x*34.0)+1.0)*x); }
    vec4 taylorInvSqrt(vec4 r) { return 1.79284291400159 - 0.85373472095314 * r; }
    
    float snoise(vec3 v) {
      const vec2 C = vec2(1.0/6.0, 1.0/3.0);
      const vec4 D = vec4(0.0, 0.5, 1.0, 2.0);
      
      // 计算网格单元格的索引
      vec3 i  = floor(v + dot(v, C.yyy));
      vec3 x0 = v - i + dot(i, C.xxx);
      
      // 计算其他三个顶点的索引
      vec3 g = step(x0.yzx, x0.xyz);
      vec3 l = 1.0 - g;
      vec3 i1 = min(g.xyz, l.zxy);
      vec3 i2 = max(g.xyz, l.zxy);
      
      // 计算相对于单元格的位置
      vec3 x1 = x0 - i1 + C.xxx;
      vec3 x2 = x0 - i2 + C.yyy;
      vec3 x3 = x0 - D.yyy;
      
      // 计算梯度
      i = mod289(i);
      vec4 p = permute(permute(permute(
                i.z + vec4(0.0, i1.z, i2.z, 1.0))
                + i.y + vec4(0.0, i1.y, i2.y, 1.0))
                + i.x + vec4(0.0, i1.x, i2.x, 1.0));
                
      // 计算梯度贡献
      float n_ = 0.142857142857;
      vec3 ns = n_ * D.wyz - D.xzx;
      
      vec4 j = p - 49.0 * floor(p * ns.z * ns.z);
      
      vec4 x_ = floor(j * ns.z);
      vec4 y_ = floor(j - 7.0 * x_);
      
      vec4 x = x_ * ns.x + ns.yyyy;
      vec4 y = y_ * ns.x + ns.yyyy;
      vec4 h = 1.0 - abs(x) - abs(y);
      
      vec4 b0 = vec4(x.xy, y.xy);
      vec4 b1 = vec4(x.zw, y.zw);
      
      vec4 s0 = floor(b0) * 2.0 + 1.0;
      vec4 s1 = floor(b1) * 2.0 + 1.0;
      vec4 sh = -step(h, vec4(0.0));
      
      vec4 a0 = b0.xzyw + s0.xzyw * sh.xxyy;
      vec4 a1 = b1.xzyw + s1.xzyw * sh.zzww;
      
      vec3 p0 = vec3(a0.xy, h.x);
      vec3 p1 = vec3(a0.zw, h.y);
      vec3 p2 = vec3(a1.xy, h.z);
      vec3 p3 = vec3(a1.zw, h.w);
      
      // 归一化梯度
      vec4 norm = taylorInvSqrt(vec4(dot(p0, p0), dot(p1, p1), dot(p2, p2), dot(p3, p3)));
      p0 *= norm.x;
      p1 *= norm.y;
      p2 *= norm.z;
      p3 *= norm.w;
      
      // 混合贡献
      vec4 m = max(0.6 - vec4(dot(x0, x0), dot(x1, x1), dot(x2, x2), dot(x3, x3)), 0.0);
      m = m * m;
      return 42.0 * dot(m * m, vec4(dot(p0, x0), dot(p1, x1), dot(p2, x2), dot(p3, x3)));
    }
    
    void main() {
      vUv = uv;
      vPosition = position;
      vNormal = normal;
      
      // 使用噪声函数创建流动的波动效果
      float noiseScale = 2.0; // 噪声缩放
      float noiseTime = time * 0.5; // 时间缩放
      float noise = snoise(vec3(position.x * noiseScale, position.y * noiseScale + noiseTime, position.z * noiseScale));
      
      // 创建流动的波动效果
      float waveStrength = 0.08; // 波动强度
      vec3 newPosition = position + normal * noise * waveStrength;
      
      // 添加网格效果 - 使用正弦函数创建网格线
      float gridSize = 0.2; // 网格大小
      float gridStrength = 0.02; // 网格强度
      float gridX = sin(position.x / gridSize + time) * gridStrength;
      float gridY = sin(position.y / gridSize + time * 1.5) * gridStrength;
      float gridZ = sin(position.z / gridSize + time * 0.7) * gridStrength;
      
      // 组合波动和网格效果
      newPosition += normal * (gridX + gridY + gridZ);
      
      gl_Position = projectionMatrix * modelViewMatrix * vec4(newPosition, 1.0);
    }
  `;
  
  const fragmentShader = `
    varying vec2 vUv;
    varying vec3 vPosition;
    varying vec3 vNormal;
    uniform float time;
    uniform vec3 glowColor;
    uniform vec3 accentColor;
    
    // 噪声函数 (简化版)
    float noise(vec3 p) {
      vec3 i = floor(p);
      vec3 f = fract(p);
      f = f * f * (3.0 - 2.0 * f);
      
      float n = i.x + i.y * 157.0 + 113.0 * i.z;
      return mix(
        mix(mix(sin(n+0.0), sin(n+1.0), f.x),
            mix(sin(n+157.0), sin(n+158.0), f.x), f.y),
        mix(mix(sin(n+113.0), sin(n+114.0), f.x),
            mix(sin(n+270.0), sin(n+271.0), f.x), f.y), f.z);
    }
    
    void main() {
      // 计算流动的光圈效果
      float flowSpeed = 0.5; // 流动速度
      float flowHeight = vPosition.y * 3.0 + time * flowSpeed;
      float glow = sin(flowHeight) * 0.5 + 0.5;
      
      // 添加噪声，使光圈更自然
      float noiseScale = 4.0;
      float noiseTime = time * 0.3;
      float noiseValue = noise(vec3(vPosition.x * noiseScale, vPosition.y * noiseScale + noiseTime, vPosition.z * noiseScale));
      glow = glow * (0.8 + noiseValue * 0.4);
      
      // 边缘发光效果 - 视角相关
      vec3 viewDirection = normalize(cameraPosition - vPosition);
      float fresnel = pow(1.0 - abs(dot(viewDirection, vNormal)), 3.0);
      
      // 网格线效果
      float gridSize = 0.1; // 网格大小
      float gridX = abs(fract(vPosition.x / gridSize + time * 0.1) - 0.5);
      float gridY = abs(fract(vPosition.y / gridSize + time * 0.15) - 0.5);
      float gridZ = abs(fract(vPosition.z / gridSize + time * 0.07) - 0.5);
      
      // 创建网格线
      float gridLine = 0.0;
      if(gridX < 0.05 || gridY < 0.05 || gridZ < 0.05) {
        gridLine = 1.0 - smoothstep(0.0, 0.05, min(min(gridX, gridY), gridZ));
      }
      
      // 组合所有效果
      vec3 baseColor = mix(glowColor, accentColor, noiseValue * 0.5);
      vec3 edgeColor = vec3(1.0, 1.0, 1.0); // 边缘白色
      vec3 gridColor = vec3(0.9, 0.9, 1.0); // 网格线颜色
      
      // 最终颜色 = 基础颜色 + 边缘发光 + 网格线
      vec3 finalColor = baseColor * glow;
      finalColor = mix(finalColor, edgeColor, fresnel * 0.7);
      finalColor = mix(finalColor, gridColor, gridLine * 0.8);
      
      // 调整透明度 - 边缘和网格线更透明
      float alpha = 0.7 * glow + fresnel * 0.3 + gridLine * 0.2;
      
      gl_FragColor = vec4(finalColor, alpha);
    }
  `;
  
  // 为每个楼层应用发光效果
  office.object.children.forEach((floor: any) => {
    floor.traverse((child: any) => {
      if (child.isMesh) {
        // 存储原始材质
        if (!child.userData) child.userData = {};
        child.userData.originalMaterial = child.material;
        
        // 创建自定义着色器材质
        const glowMaterial = new THREE.ShaderMaterial({
          uniforms: {
            time: { value: 0.0 },
            glowColor: { value: new THREE.Color(0x0066ff) }, // 主要发光颜色
            accentColor: { value: new THREE.Color(0x00ffcc) }, // 辅助发光颜色
            cameraPosition: { value: viewer.camera.position }
          },
          vertexShader: vertexShader,
          fragmentShader: fragmentShader,
          transparent: true,
          side: THREE.DoubleSide
        });
        
        // 应用新材质
        child.material = glowMaterial;
      }
    });
  });
  
  // 创建动画更新函数
  const updateGlowEffect = () => {
    const time = performance.now() * 0.001; // 转换为秒
    
    office.object.traverse((child: any) => {
      if (child.isMesh && child.material.uniforms) {
        child.material.uniforms.time.value = time;
        child.material.uniforms.cameraPosition.value = viewer.camera.position;
      }
    });
  };
  
  // 添加到动画循环
  glowEffect = {
    fun: updateGlowEffect,
    content: viewer
  };
  
  viewer.addAnimate(glowEffect);
};

// 移除发光效果
const removeGlowEffect = () => {
  if (!office) return;
  
  // 恢复原始材质
  office.object.traverse((child: any) => {
    if (child.isMesh && child.userData && child.userData.originalMaterial) {
      child.material = child.userData.originalMaterial;
    }
  });
  
  // 移除动画更新
  if (glowEffect && viewer) {
    // 从动画列表中移除
    const index = viewer.animateEventList.findIndex(item => item === glowEffect);
    if (index !== -1) {
      viewer.animateEventList.splice(index, 1);
    }
    glowEffect = null;
  }
};

</script>

<style scoped>
#three {
  height: 100%;
  width: 100%;
}

.control-buttons {
  position: absolute;
  top: 20px;
  left: 20px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  z-index: 100;
}

.control-btn {
  background-color: rgba(29, 78, 216, 0.8);
  color: white;
  border: none;
  padding: 10px 15px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
  transition: background-color 0.3s;
  width: 100px;
  text-align: center;
}

.control-btn:hover {
  background-color: rgba(30, 64, 175, 0.9);
}
</style>

<script lang="ts">
export default {
  name: 'Sence'
};
</script>
