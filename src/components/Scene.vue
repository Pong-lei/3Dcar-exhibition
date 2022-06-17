<template>
  <div>
    <div class="container" ref="container"></div>
  </div>
</template>

<script setup>
import * as THREE from "three";
import Dat from "dat.gui";
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
const gui = new Dat.GUI();
const container = ref(null);
let controls = ref(null);

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

const gltfLoader = new GLTFLoader();
const dracoLoader = new DRACOLoader();
let loadFalg = false;
dracoLoader.setDecoderPath("/public/draco/");
gltfLoader.setDRACOLoader(dracoLoader);
gltfLoader.load(
  "/models/terzo.gltf",
  (gltf) => {
    gltf.scene.rotation.y = Math.PI / 2;
    scene.add(gltf.scene);
    loadFalg = true;
  },
  (xh) => {
    // console.log(xh);
  }
);

/**
 *
 * Lights
 *
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
gui.add(directionalLight1, "intensity", 1, 25, 0.5);

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
    bloomStrength: 3.79,
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


  const controller = {
    radius: halftonePass.uniforms["radius"].value,
    rotateR: halftonePass.uniforms["rotateR"].value / (Math.PI / 180),
    rotateG: halftonePass.uniforms["rotateG"].value / (Math.PI / 180),
    rotateB: halftonePass.uniforms["rotateB"].value / (Math.PI / 180),
    scatter: halftonePass.uniforms["scatter"].value,
    shape: halftonePass.uniforms["shape"].value,
    greyscale: halftonePass.uniforms["greyscale"].value,
    blending: halftonePass.uniforms["blending"].value,
    blendingMode: halftonePass.uniforms["blendingMode"].value,
    disable: halftonePass.uniforms["disable"].value,
  };

  function onGUIChange() {
    // update uniforms
    halftonePass.uniforms["radius"].value = controller.radius;
    halftonePass.uniforms["rotateR"].value =
      controller.rotateR * (Math.PI / 180);
    halftonePass.uniforms["rotateG"].value =
      controller.rotateG * (Math.PI / 180);
    halftonePass.uniforms["rotateB"].value =
      controller.rotateB * (Math.PI / 180);
    halftonePass.uniforms["scatter"].value = controller.scatter;
    halftonePass.uniforms["shape"].value = controller.shape;
    halftonePass.uniforms["greyscale"].value = controller.greyscale;
    halftonePass.uniforms["blending"].value = controller.blending;
    halftonePass.uniforms["blendingMode"].value = controller.blendingMode;
    halftonePass.uniforms["disable"].value = controller.disable;
  }

  gui
    .add(controller, "shape", { Dot: 1, Ellipse: 2, Line: 3, Square: 4 })
    .onChange(onGUIChange);
  gui.add(controller, "radius", 1, 25).onChange(onGUIChange);
  gui.add(controller, "rotateR", 0, 90).onChange(onGUIChange);
  gui.add(controller, "rotateG", 0, 90).onChange(onGUIChange);
  gui.add(controller, "rotateB", 0, 90).onChange(onGUIChange);
  gui.add(controller, "scatter", 0, 1, 0.01).onChange(onGUIChange);
  gui.add(controller, "greyscale").onChange(onGUIChange);
  gui.add(controller, "blending", 0, 1, 0.01).onChange(onGUIChange);
  gui
    .add(controller, "blendingMode", {
      Linear: 1,
      Multiply: 2,
      Add: 3,
      Lighter: 4,
      Darker: 5,
    })
    .onChange(onGUIChange);
  gui.add(controller, "disable").onChange(onGUIChange);

  gui
    .add(params, "bloomThreshold", 0.0, 1.0)
    .step(0.01)
    .name("阈值")
    .onChange(function (value) {
      bloomPass.threshold = Number(value);
    });

  // 强度 在0-10之间可正常看到物体，超过10会因光线过强而看不见物体，步长建议0.01
  gui
    .add(params, "bloomStrength", 0, 10)
    .step(0.01)
    .name("强度")
    .onChange(function (value) {
      bloomPass.strength = Number(value);
    });

  gui
    .add(params, "bloomRadius", 0.0, 1.0)
    .step(0.01)
    .name("半径")
    .onChange(function (value) {
      bloomPass.radius = Number(value);
    });

  return { finalComposer, glowComposer };
}

const { finalComposer, glowComposer } = initComposer();

if (!renderer.capabilities.isWebGL2) {
  let smaaPass = new SMAAPass();
  finalComposer.addPass(smaaPass);
}

// animate
const render = () => {
  if (loadFalg) {
    const elapsedTime = clock.getElapsedTime();

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
  // Setup camera controller
  controls = new OrbitControls(camera, container.value);
  controls.enableDaming = true;
  controls.dampingFactor = 110;
  container.value.appendChild(renderer.domElement);
  render();
});
</script>

<style lang="less">
* {
  margin: 0px;
  padding: 0px;
}

.container {
  height: 100vh;
  width: 100vw;
}
</style>
