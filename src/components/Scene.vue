<template>
  <div>
    <div class="container" ref="container"></div>
    <div class="point ponint0">
      <div class="label">1</div>
      <div class="text">
        Lorem ipsum dolor sit amet consectetur, adipisicing elit.
      </div>
    </div>
    <div class="point ponint1">
      <div class="label">2</div>
      <div class="text">
        Lorem ipsum dolor sit amet consectetur, adipisicing elit.
      </div>
    </div>
    <div class="point ponint2">
      <div class="label">3</div>
      <div class="text">
        Lorem ipsum dolor sit amet consectetur, adipisicing elit.
      </div>
    </div>
  </div>
</template>

<script setup>
import * as THREE from "three";
import Stats from "stats.js";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer";
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass";
import { ShaderPass } from "three/examples/jsm/postprocessing/ShaderPass";
import { RGBShiftShader } from "three/examples/jsm/shaders/RGBShiftShader";
import { SMAAPass } from "three/examples/jsm/postprocessing/SMAAPass";
import { UnrealBloomPass } from "three/examples/jsm/postprocessing/UnrealBloomPass.js";
import { HalftonePass } from "three/examples/jsm/postprocessing/HalftonePass.js";
import { ref, onMounted } from "vue";

/**
 *
 * debug
 *
 * */

const container = ref(null);
let points = [];
let state = new Stats();
let controlsFlag = false;
let loadFalg = false;

let glows = [
  "Object_11",
  "Object_159",
  "Object_67",
  "Object_74",
  "Object_25",
  "Object_72",
  "Object_73",
  "Object_120",
  "Object_133",
  "Object_70",
  "Object_203",
  "Object_197",
  "Object_172",
  "Object_186",
  "Object_86",
  "Object_160",
  "Object_82",
];
// Setup your scene
const scene = new THREE.Scene();
const clock = new THREE.Clock();

// Setup a camera
const camera = new THREE.PerspectiveCamera(
  70,
  window.innerWidth / window.innerHeight,
  0.1,
  1000
);
camera.position.set(3, 2.5, 2.5);
// camera.lookAt(new THREE.Vector3());
function onWindowResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
  glowComposer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  finalComposer.setSize(window.innerWidth, window.innerHeight);
}
window.addEventListener("resize", onWindowResize);

const settingLayer = () => {
  scene.traverse((ch) => {
    if (ch.isMesh && glows.includes(ch.name)) {
      ch.layers.set(1);
    } else {
      ch.layers.set(0);
    }
  });
};

const gltfLoader = new GLTFLoader();
const dracoLoader = new DRACOLoader();

dracoLoader.setDecoderPath("/public/draco/");
gltfLoader.setDRACOLoader(dracoLoader);
gltfLoader.load("/models/terzo.gltf", (gltf) => {
  gltf.scene.rotation.y = Math.PI / 2;
  scene.add(gltf.scene);
  // settingLayer();
  loadFalg = true;
});
/**
 * Points of insterest
 * */
const rayCaster = new THREE.Raycaster();
const initPoints = () => {
  points = [
    {
      position: new THREE.Vector3(1, 0.6, 0.98),
      el: document.querySelector(".ponint0"),
    },
    {
      position: new THREE.Vector3(-1.2, 0.9, -0.6),
      el: document.querySelector(".ponint1"),
    },
    {
      position: new THREE.Vector3(2.1, 0.33, -0.77),
      el: document.querySelector(".ponint2"),
    },
  ];
};

/**
 * Lights
 */
const directionalLight1 = new THREE.DirectionalLight(0xffffff, 7);
const directionalLight2 = new THREE.DirectionalLight(0xffffff, 2);
const directionalLight3 = new THREE.DirectionalLight(0xffffff, 2);
// const AmbientLight = new THREE.AmbientLight(0xffffff, 2);
// directionalLight.castShadow = true;
directionalLight1.position.set(0, 3, 0);
directionalLight2.position.set(2, 0, 0);
directionalLight3.position.set(-2, 0, 0);

scene.add(directionalLight1);
scene.add(directionalLight2);
scene.add(directionalLight3);
// scene.add(AmbientLight);

const renderer = new THREE.WebGLRenderer({
  antialias: true,
});
renderer.setClearColor(0x111111, 1);
renderer.physicallyCorrectLights = true;
// renderer.outputEncoding = THREE.sRGBEncoding;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExpose = 2;
renderer.shadowMap.enable = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

// Setup camera controller
let controls = new OrbitControls(camera, renderer.domElement);
controls.enableDaming = true;
controls.dampingFactor = 0.2;
controls.panSpeed = 0.5;
controls.maxDistance = 4; // 最大缩放距离
controls.rotateSpeed = 0.5;

function darkenMaterial(obj) {
  let name = obj.name;
  if (obj.isMesh && !glows.includes(name)) {
    const material = obj.material;
    obj.originalMaterial = obj.material;
    const Proto = Object.getPrototypeOf(material).constructor;
    obj.material = new Proto({ color: new THREE.Color(0x000000) });
  }
}

/**
 * 还原材质
 */
function restoreMaterial(obj) {
  if (!obj.originalMaterial) return;
  obj.material = obj.originalMaterial;
  delete obj.originalMaterial;
}

const renderTarget = new THREE.WebGLRenderTarget(
  window.innerWidth,
  window.innerHeight,
  {
    minFilter: THREE.LinearFilter,
    magFilter: THREE.LinearFilter,
    format: THREE.RGBAFormat,
    encoding: THREE.sRGBEncoding,
    samples: 10,
  }
);

function initComposer() {
  // 辉光参数
  const params = {
    // 强度
    bloomStrength: 2.7,
    // 阈值
    bloomThreshold: 0.29,
    // 半径
    bloomRadius: 1,
  };
  const HalfToneParams = {
    shape: 1,
    radius: 3.4,
    rotateR: Math.PI / 12,
    rotateB: (Math.PI / 12) * 2,
    rotateG: (Math.PI / 12) * 3,
    scatter: 1,
    blending: 0.1,
    blendingMode: 1,
    greyscale: false,
    disable: false,
  };
  const halftonePass = new HalftonePass(
    window.innerWidth,
    window.innerHeight,
    HalfToneParams
  );

  // 渲染通道
  const renderPass = new RenderPass(scene, camera);
  // 辉光通道
  const bloomPass = new UnrealBloomPass(
    new THREE.Vector2(800, 600),
    1.5,
    0.4,
    0.85
  );

  bloomPass.threshold = params.bloomThreshold;
  bloomPass.strength = params.bloomStrength;
  bloomPass.radius = params.bloomRadius;

  // 辉光合成器
  const glowComposer = new EffectComposer(renderer, renderTarget);
  glowComposer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  glowComposer.renderToScreen = false;
  glowComposer.addPass(renderPass);
  glowComposer.addPass(bloomPass);

  // 最终通道
  const finalPass = new ShaderPass(
    new THREE.ShaderMaterial({
      uniforms: {
        baseTexture: { value: null },
        bloomTexture: { value: glowComposer.renderTarget2.texture },
      },
      vertexShader: `
      varying vec2 vUv;
			void main() {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
      }`,
      fragmentShader: `
      uniform sampler2D baseTexture;
			uniform sampler2D bloomTexture;

			varying vec2 vUv;
			void main() {
				gl_FragColor = ( texture2D( baseTexture, vUv ) + vec4( 1.0 ) * texture2D( bloomTexture, vUv ) );
			}`,
      defines: {},
    }),
    "baseTexture"
  );
  finalPass.needsSwap = true;
  // 最终合成器
  const finalComposer = new EffectComposer(renderer, renderTarget);
  finalComposer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  finalComposer.addPass(renderPass);
  finalComposer.addPass(finalPass);
  finalComposer.addPass(halftonePass);

  return { finalComposer, glowComposer };
}

const { finalComposer, glowComposer } = initComposer();

if (!renderer.capabilities.isWebGL2) {
  let smaaPass = new SMAAPass();
  finalComposer.addPass(smaaPass);
}

// animate
const render = () => {
  state.update();
  if (loadFalg) {
    const elapsedTime = clock.getElapsedTime();
    for (const point of points) {
      const screenPosition = point.position.clone();
      screenPosition.project(camera);

      const translateX = screenPosition.x * window.innerWidth * 0.5;
      const translateY = -screenPosition.y * window.innerHeight * 0.5;

      point.el.style.transform = `translateX(${translateX}px) translateY(${translateY}px)`;
      if (!controlsFlag) {
        rayCaster.setFromCamera(screenPosition, camera);
        const instersects = rayCaster.intersectObjects(scene.children, true);
        if (instersects.length === 0) {
          point.el.classList.add("visible");
        } else {
          const instersectsDistance = instersects[0].distance;
          const pointDistance = point.position.distanceTo(camera.position);
          if (instersectsDistance < pointDistance) {
            point.el.classList.remove("visible");
          } else {
            point.el.classList.add("visible");
          }
        }
      } else {
        point.el.classList.remove("visible");
      }
    }
    renderer.render(scene, camera);
    // 不辉光的先变黑
    scene.traverse(darkenMaterial);
    // 渲染辉光合成器
    glowComposer.render();
    // 还原不会光的材质
    scene.traverse(restoreMaterial);
    // 渲染最终合成器
    finalComposer.render();
    controls.update();
    // renderer.render(scene, camera);
  }

  requestAnimationFrame(render);
};
onMounted(() => {
  initPoints();
  document.body.appendChild(state.dom);
  container.value.addEventListener("mousedown", () => {
    controlsFlag = true;
  });
  container.value.addEventListener("mouseup", () => {
    controlsFlag = false;
  });
  container.value.appendChild(renderer.domElement);
  render();
});
</script>

<style lang="less">
.container {
  height: 100vh;
  width: 100vw;
}
.point {
  position: absolute;
  top: 50%;
  left: 50%;
  color: #ffffff;
  font-family: Helvetica, Arial, sans-serif;
  font-weight: 100;
  font-size: 12px;

  .label {
    position: absolute;
    top: -10px;
    left: -10px;
    width: 20px;
    height: 20px;
    background: #00000077;
    border-radius: 50%;
    text-align: center;
    line-height: 20px;
    cursor: help;
    transform: scale(0, 0);
    transition: transform 0.3s;
  }
  .text {
    opacity: 0;
    position: absolute;
    top: 20px;
    left: -90px;
    padding: 10px;
    border-radius: 4px;
    width: 160px;
    background: #00000077;
    line-height: 1.3em;
    text-align: center;
    transition: opacity 0.3s;
    pointer-events: none;
  }
  &:hover .text {
    opacity: 1;
  }
  &.visible .label {
    transform: scale(1.1);
  }
}
</style>
