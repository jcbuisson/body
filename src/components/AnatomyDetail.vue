<template>
   <main class="container max-w-7xl bg-slate-50" style="position: relative;">

      <div class="flex w-full gap-2">
         <button class="rounded-xl karan-btn px-6" @click="resetMeshes">
            Effacer
         </button>

         <button class="rounded-xl karan-btn px-6" @click="resetView">
            Recentrer
         </button>

         <!-- search input -->
         <label class="input input-bordered flex items-center gap-2 my-2" style="position: relative; z-index: 50;">
            <input v-model="searchTerms" type="text" class="grow" placeholder="Rechercher..." />
            <svg
               xmlns="http://www.w3.org/2000/svg"
               viewBox="0 0 16 16"
               fill="currentColor"
               class="h-6 w-6 opacity-70">
               <path
                  fill-rule="evenodd"
                  d="M9.965 11.026a5 5 0 1 1 1.06-1.06l2.755 2.754a.75.75 0 1 1-1.06 1.06l-2.755-2.754ZM10.5 7a3.5 3.5 0 1 1-7 0 3.5 3.5 0 0 1 7 0Z"
                  clip-rule="evenodd" />
            </svg>
         </label>
      </div>

      <!-- search results menu -->
      <ul class="menu bg-slate-200 rounded-box w-100 absolute z-50" v-if="searchedMeshNameList.length">
         <template v-for="mesh in searchedMeshNameList">
            <li><a @click="selectMeshName(mesh)">{{ mesh }}</a></li>
         </template>
      </ul>

      <!-- loading overlay -->
      <div v-if="isLoading" class="absolute inset-0 flex flex-col items-center justify-center bg-slate-50 z-40">
         <div class="w-14 h-14 rounded-full border-4 border-slate-200 border-t-violet-500 animate-spin mb-4"></div>
         <span class="text-slate-500 text-sm">Chargement {{ loadingProgress }}%</span>
      </div>

      <!-- 3D viewer -->
      <div ref="container" class="viewer"></div>

      <!-- mesh info sticker -->
      <div ref="meshInfo" class="meshInfo"></div>
   </main>
</template>

<script setup>
import { onMounted, ref, computed, watch, onBeforeUnmount } from "vue";
import * as THREE from "three";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js"
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

// import { getAnatomy, anatomyOfId } from '/src/use/useAnatomy'

import meshKey2Name from '/src/assets/meshKey2Name.json'


const props = defineProps({
   anatomy_id: {
      type: Number,
      required: true
   },
})


// const anatomy = anatomyOfId.value(props.anatomy_id);

const container = ref(null)
const loadingProgress = ref(0)
const isLoading = ref(true)
const searchTerms = ref('')
const meshInfo = ref(null)

let scene, camera, renderer, controls, raycaster, mouse, model, animationId;
const INFO_TIMEOUT = 1500;

onMounted(async () => {
   initScene()
   await loadModel()
   animate()
   window.addEventListener("resize", onResize)
   renderer.domElement.addEventListener("pointerdown", onPointerDown)
   renderer.domElement.addEventListener("pointerup", onPointerUp)
   renderer.domElement.addEventListener("pointermove", onPointerMove)
   renderer.domElement.addEventListener("wheel", onZoom)
});

onBeforeUnmount(() => {
   window.removeEventListener("resize", onResize);
   if (renderer) {
      renderer.domElement.removeEventListener("pointerdown", onPointerDown);
      renderer.domElement.removeEventListener("pointerup", onPointerUp);
      renderer.domElement.removeEventListener("pointermove", onPointerMove);
      renderer.domElement.removeEventListener("wheel", onZoom);
      renderer.dispose();
   }
   cancelAnimationFrame(animationId);
});

let INITIAL_CAMERA_POSITION;
const INITIAL_CONTROLS_TARGET = new THREE.Vector3(0, 0, 0);

function resetView() {
   camera.position.copy(INITIAL_CAMERA_POSITION);
   controls.target.copy(INITIAL_CONTROLS_TARGET);
   controls.update();
}

function initScene() {
   scene = new THREE.Scene();
   scene.background = new THREE.Color(0xf0f0f0);

   camera = new THREE.PerspectiveCamera(
      75,
      container.value.clientWidth / container.value.clientHeight,
      0.1,
      1000
   )
   camera.position.set(0, 1.5, 3)
   INITIAL_CAMERA_POSITION = camera.position.clone();

   renderer = new THREE.WebGLRenderer({ antialias: true });
   renderer.setSize(container.value.clientWidth, container.value.clientHeight);
   container.value.appendChild(renderer.domElement)

   // Contrôles orbitaux
   controls = new OrbitControls(camera, renderer.domElement);
   controls.enableDamping = true;
   controls.dampingFactor = 0.05;

   // Lumière ambiante
   const ambientLight = new THREE.AmbientLight(0xffffff, 0.8);
   scene.add(ambientLight);

   // Lumière directionnelle
   const directionalLight = new THREE.DirectionalLight(0xffffff, 0.6);
   directionalLight.position.set(1, 1, 1).normalize();
   scene.add(directionalLight);

   raycaster = new THREE.Raycaster();
   mouse = new THREE.Vector2();
}

async function loadModel() {
   const loader = new GLTFLoader()
   // const anatomy = await getAnatomy(props.anatomy_id)
   // const filePath = `/static/uploads/${anatomy?.content}`
   const filePath = `/static/uploads/arthrologie.glb`
   loader.load(filePath,
      (gltf) => {
         model = gltf.scene;
         scene.add(model);
         isLoading.value = false;
      },
      (xhr) => {
         if (xhr.total) {
            loadingProgress.value = Math.round((xhr.loaded / xhr.total) * 100);
         } else {
            loadingProgress.value = 0;
         }
      },
      (error) => {
         console.error("Error loading model:", error);
         isLoading.value = false;
      }
   )
}

function nomalizeString(str) {
   return str
      .toLowerCase()
      // retire _, espaces, - 
      .replace(/[_\s\-]+/g, '')
      // retire suffixes côté (l/r/left/right en fin)
      .replace(/(left|right|_l\b|_r\b|\bl\b|\br\b)$/g, '')
      // retire les chiffres
      .replace(/\d+/g, '')
      // retire accents
      .normalize('NFD').replace(/[\u0300-\u036f]/g, '');
}

const searchedMeshNameList = computed(() => {
   if (searchTerms.value.trim().length === 0) return [];
   const searchStr = searchTerms.value.trim().toLowerCase();
   const result = [];
   model.traverse((obj) => {
      if (obj.isMesh) {
         // WARNING: obj.name is normalized or not, depending on the model
         const meshName = meshKey2Name[obj.name] || meshKey2Name[nomalizeString(obj.name)];
         if (meshName) {
            if (meshName.toLowerCase().includes(searchStr)) {
               if (!result.includes(meshName)) result.push(meshName);
            }
         }
      }
   });
   return result.sort();
})

function selectMeshName(name) {
   const matchingKeys = Object.entries(meshKey2Name).filter(([meshKey, meshName]) => name === meshName).map(entry => entry[0]);
   searchTerms.value = '';

   // Highlight matching meshes
   model.traverse((obj) => {
      if (obj.isMesh) {
         // WARNING: obj.name is normalized or not, depending on the model
         if (matchingKeys.includes(obj.name) || matchingKeys.includes(nomalizeString(obj.name))) {
            highlightMesh(obj);
            if (isMeshTransparent(obj)) setMeshVisible(obj);
         }
      }
   });
}

// const HIGHLIGHT_MATERIAL = new THREE.MeshStandardMaterial({ color: 0xff0000 });
const HIGHLIGHT_COLOR = 0x39FF14; // vert fluo bien visible
const HIGHLIGHT_MATERIAL = new THREE.MeshStandardMaterial({
   color: HIGHLIGHT_COLOR,
   emissive: HIGHLIGHT_COLOR,
   emissiveIntensity: 0.6,
   metalness: 0.2,
   roughness: 0.4
});

function isMeshHighlighted(mesh) {
   return !!mesh.userData.isHighlighted;
}

function highlightMesh(mesh) {
   if (!mesh.userData.originalMaterial) mesh.userData.originalMaterial = mesh.material;
   mesh.material = HIGHLIGHT_MATERIAL;
   mesh.userData.isHighlighted = true;
}

function unhighlightMesh(mesh) {
   mesh.material = mesh.userData.originalMaterial;
   mesh.userData.isHighlighted = false;
}

function setMeshTransparent(mesh) {
   if (!mesh.userData.originalMaterial) mesh.userData.originalMaterial = mesh.material;
   const transparentMaterial = mesh.material.clone();
   transparentMaterial.transparent = true;
   transparentMaterial.opacity = 0.05;
   transparentMaterial.depthWrite = false; // permet les clics à travers
   mesh.material = transparentMaterial;
   mesh.userData.isTransparent = true;
}

function setMeshVisible(mesh) {
   mesh.material = mesh.userData.originalMaterial;
   mesh.userData.isTransparent = false;
}

function isMeshTransparent(mesh) {
   return !!mesh.userData.isTransparent;
}

// display mesh name near mouse click, for duration INFO_TIMEOUT
async function displayMeshInfo(mesh, mouseX, mouseY) {
   // WARNING: mesh.name is normalized or not, depending on the model
   const name = meshKey2Name[mesh.name] || meshKey2Name[nomalizeString(mesh.name)] || mesh.name;
   meshInfo.value.textContent = name;
   meshInfo.value.style.left = `${mouseX - 10}px`;
   meshInfo.value.style.top = `${mouseY - 80}px`;
   meshInfo.value.style.display = 'block';
}

function clearMeshInfo() {
   meshInfo.value.style.display = 'none';
}

const resetMeshes = () => {
   model.traverse((obj) => {
      if (obj.isMesh) {
         if (isMeshHighlighted(obj)) unhighlightMesh(obj);
         if (isMeshTransparent(obj)) setMeshVisible(obj);
      }
   });
   clearMeshInfo();
}

function getFirstIntersectedMesh(clientX, clientY) {
   if (!camera || !model) return;
   const rect = renderer.domElement.getBoundingClientRect();
   mouse.x = ((clientX - rect.left) / rect.width) * 2 - 1;
   mouse.y = -((clientY - rect.top) / rect.height) * 2 + 1;
   raycaster.setFromCamera(mouse, camera);
   const intersectedObjects = raycaster.intersectObjects(model.children, true);
   const intersectedMeshes = intersectedObjects.map(obj => obj.object);
   const visibleIntersectedMeshes = intersectedMeshes.filter(mesh => !isMeshTransparent(mesh));
   return (visibleIntersectedMeshes.length === 0) ? undefined : visibleIntersectedMeshes[0];
}


//////////////////      ABSTRACT MOUSE ACTVITY INTO CALLS TO: onSingleClick(event), onDoubleClick(event), onPanRotate(event)     /////////////////

let panRotateZoomTimer = null;
let doubleClickTimer = null;
const PAN_ROTATE_ZOOM_TIMEOUT = 500;
const DOUBLE_CLICK_TIMEOUT = 200;

let mouseState = 'idle'; // 'idle', 'select', 'pan-rotate'
let clickCount = 0;

function onPointerDown(event) {
   mouseState = 'select';
   if (!panRotateZoomTimer) {
      // rotation/pan/zoom is considered as such after PAN_ROTATE_ZOOM_TIMEOUT
      panRotateZoomTimer = setTimeout(() => {
         panRotateZoomTimer = null;
         mouseState = 'pan-rotate';
      }, PAN_ROTATE_ZOOM_TIMEOUT);
   }
}

function onPointerUp(event) {
   clearTimeout(panRotateZoomTimer);
   panRotateZoomTimer = null;
   clickCount += 1;
   if (!doubleClickTimer) {
      doubleClickTimer = setTimeout(async () => {
         // console.log('clickCount', clickCount, 'mouseState', mouseState);
         if (mouseState === 'select') {
            if (clickCount === 1) {
               onSingleClick(event)
            } else if (clickCount > 1) {
               onDoubleClick(event)
            }
         } else if (mouseState === 'pan-rotate') {
            onPanRotate(event)
         }

         clickCount = 0;
         doubleClickTimer = null;

         mouseState = 'idle';
      }, DOUBLE_CLICK_TIMEOUT);
   }
}

function onPointerMove(event) {
   if (mouseState === 'pan-rotate') {
      onPanRotate(event)
   }
}

// function onZoom(event) {
// }
///////////////////////////////////


async function onSingleClick(event) {
   console.log('single click!');
   const mesh = getFirstIntersectedMesh(event.clientX, event.clientY);
   console.log('mesh', mesh?.name);
   if (mesh) {
      // select a mesh
      if (isMeshHighlighted(mesh)) {
         unhighlightMesh(mesh);
         clearMeshInfo();
      } else {
         highlightMesh(mesh);
         await displayMeshInfo(mesh, event.clientX, event.clientY);
      }
   } else {
      // select empty space
      clearMeshInfo();
   }
}

function onDoubleClick(event) {
   console.log('double click!');
   const mesh = getFirstIntersectedMesh(event.clientX, event.clientY);
   console.log('mesh', mesh?.name);
   if (mesh) {
      setMeshTransparent(mesh);
   }
   clearMeshInfo();
}

function onPanRotate(event) {
   console.log('onPanRotate!');
   clearMeshInfo();
}

function onZoom(event) {
   clearMeshInfo();
}

function onResize() {
   if (!renderer || !camera) return;
   camera.aspect = container.value.clientWidth / container.value.clientHeight;
   camera.updateProjectionMatrix();
   renderer.setSize(container.value.clientWidth, container.value.clientHeight);
}

function animate() {
   animationId = requestAnimationFrame(animate);
   renderer.render(scene, camera);
}
</script>

<style>
.viewer {
   width: 100%;
   height: 100vh;
   overflow: hidden;
   position: relative;
}
</style>

<style scoped>
.meshInfo {
   position: absolute;
   top: 10px;
   left: 10px;
   background: rgba(0, 0, 0, 0.7);
   color: white;
   padding: 8px;
   border-radius: 5px;
   display: none;
}
</style>