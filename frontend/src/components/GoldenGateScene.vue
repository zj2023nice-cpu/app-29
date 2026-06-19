<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref, computed } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { Water } from 'three/examples/jsm/objects/Water.js';
import { Sky } from 'three/examples/jsm/objects/Sky.js';

const canvasContainer = ref<HTMLDivElement | null>(null);

// Configuration
const BRIDGE_COLOR = 0xF04A00;
const ROAD_COLOR = 0x333333;
const CABLE_COLOR = 0xF04A00;
const SF_LATITUDE = 37.7749 * Math.PI / 180;
const SUN_DECLINATION = 20.0 * Math.PI / 180;
const TIME_SPEED = { NORMAL: 1, FAST: 10, SUPER_FAST: 100 } as const;
const PRESET_TIMES = {
  SUNRISE: 6 / 24,
  NOON: 12 / 24,
  SUNSET: 18 / 24,
  MIDNIGHT: 0
} as const;

let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let water: Water;
let sky: Sky;
let sun: THREE.Vector3;
let sunLight: THREE.DirectionalLight;
let ambientLight: THREE.AmbientLight;
let pmremGenerator: THREE.PMREMGenerator;
let sceneEnv: THREE.Scene;
let renderTarget: THREE.WebGLRenderTarget | null;
let bridgeLights: THREE.PointLight[] = [];
let bridgeLightMeshes: THREE.Mesh[] = [];
let animationId: number;
let lastFrameTime: number = 0;

const timeOfDay = ref<number>(PRESET_TIMES.NOON);
const isPlaying = ref<boolean>(false);
const speedMultiplier = ref<number>(TIME_SPEED.NORMAL);

const formattedTime = computed<string>(() => {
  const totalMinutes = Math.floor(timeOfDay.value * 24 * 60);
  const hours = Math.floor(totalMinutes / 60) % 24;
  const minutes = totalMinutes % 60;
  return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}`;
});

const cleanUp = (): void => {
  if (animationId) cancelAnimationFrame(animationId);
  if (renderer) renderer.dispose();
  if (controls) controls.dispose();
  if (pmremGenerator) pmremGenerator.dispose();
  window.removeEventListener('resize', onWindowResize);
};

const onWindowResize = (): void => {
  if (!camera || !renderer) return;
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
};

// Three.js 坐标系约定：+x = 东，+z = 北，+y = 天顶（向上）
// 金门大桥沿 x 轴（东西方向）横跨，两座桥塔在 ±x 位置
// 旧金山（北纬 37.77°）夏半年太阳轨迹：
//   日出：东北方向 (x+, z+)
//   正午：正南方向 (x=0, z-)，太阳高度最高
//   日落：西北方向 (x-, z+)
//   午夜：正北方向地平线以下 (x=0, z+, y<0)
const calculateSunDirection = (t: number): THREE.Vector3 => {
  const H = (t * 24 - 12) * 15 * Math.PI / 180;
  const lat = SF_LATITUDE;
  const dec = SUN_DECLINATION;

  const cosDec = Math.cos(dec);
  const sinDec = Math.sin(dec);
  const cosLat = Math.cos(lat);
  const sinLat = Math.sin(lat);
  const cosH = Math.cos(H);
  const sinH = Math.sin(H);

  const x = -cosDec * sinH;
  const z = cosLat * sinDec - sinLat * cosDec * cosH;
  const y = sinLat * sinDec + cosLat * cosDec * cosH;

  return new THREE.Vector3(x, y, z).normalize();
};

const smoothstep = (edge0: number, edge1: number, x: number): number => {
  const t = Math.max(0, Math.min(1, (x - edge0) / (edge1 - edge0)));
  return t * t * (3 - 2 * t);
};

const lerpColor = (color1: THREE.Color, color2: THREE.Color, t: number): THREE.Color => {
  return new THREE.Color().lerpColors(color1, color2, t);
};

const refreshEnvironmentMap = (): void => {
  if (!sky || !pmremGenerator || !sceneEnv) return;
  if (renderTarget) renderTarget.dispose();
  sceneEnv.add(sky);
  renderTarget = pmremGenerator.fromScene(sceneEnv);
  scene.add(sky);
  scene.environment = renderTarget.texture;
};

const updateSceneForTime = (t: number): void => {
  const sunDir = calculateSunDirection(t);
  const sunDistance = 4500;
  sun.copy(sunDir).multiplyScalar(sunDistance);
  const sunHeight = sunDir.y;
  const elevation = Math.asin(Math.max(-1, Math.min(1, sunHeight)));

  if (sky) {
    const skyMat = sky.material as THREE.ShaderMaterial;
    skyMat.uniforms['sunPosition']!.value.copy(sun);
    skyMat.uniforms['turbidity']!.value = 8 + smoothstep(-0.1, 0.3, elevation) * 4;
    skyMat.uniforms['rayleigh']!.value = 0.5 + smoothstep(-0.1, 0.5, elevation) * 2.5;
  }

  if (water) {
    const waterMat = water.material as THREE.ShaderMaterial;
    waterMat.uniforms['sunDirection']!.value.copy(sunDir);
  }

  const dayFactor = smoothstep(-0.05, 0.15, sunHeight);
  const nightFactor = 1 - dayFactor;
  const twilightFactor = smoothstep(-0.15, 0, sunHeight) * (1 - smoothstep(0, 0.15, sunHeight));

  const nightColor = new THREE.Color(0x0a0a1a);
  const twilightColor = new THREE.Color(0xff6b35);
  const sunriseColor = new THREE.Color(0xffd4a3);
  const dayColor = new THREE.Color(0x87ceeb);

  let skyColor: THREE.Color;
  if (sunHeight < 0) {
    skyColor = lerpColor(nightColor, twilightColor, smoothstep(-0.3, -0.05, sunHeight));
  } else {
    const dayBlend = smoothstep(0, 0.3, sunHeight);
    skyColor = lerpColor(lerpColor(twilightColor, sunriseColor, dayBlend), dayColor, dayBlend);
  }

  scene.background = skyColor;

  ambientLight.intensity = 0.05 + dayFactor * 0.45;
  ambientLight.color.copy(lerpColor(new THREE.Color(0x1a1a3a), new THREE.Color(0xcccccc), dayFactor));

  const sunIntensity = Math.max(0, sunHeight) * 2.5;
  const sunCol = sunHeight < 0.1
    ? lerpColor(new THREE.Color(0xff4400), new THREE.Color(0xffaa33), smoothstep(-0.05, 0.1, sunHeight))
    : lerpColor(new THREE.Color(0xffaa33), new THREE.Color(0xffffee), smoothstep(0.1, 0.6, sunHeight));
  sunLight.intensity = sunIntensity;
  sunLight.color.copy(sunCol);
  sunLight.position.copy(sun);

  renderer.toneMappingExposure = 0.2 + dayFactor * 0.5;

  if (water) {
    (water.material as THREE.ShaderMaterial).uniforms['waterColor']!.value.copy(
      lerpColor(new THREE.Color(0x000510), new THREE.Color(0x0044aa), dayFactor)
    );
  }

  if (scene.fog) {
    const fog = scene.fog as THREE.FogExp2;
    fog.density = 0.0005 + nightFactor * 0.0025 + twilightFactor * 0.002;
    fog.color.copy(lerpColor(new THREE.Color(0x0a0a1a), skyColor, dayFactor));
  }

  const lightIntensity = nightFactor + twilightFactor * 0.5;
  bridgeLights.forEach((light) => { light.intensity = lightIntensity * 2; });
  bridgeLightMeshes.forEach((mesh) => {
    const mat = mesh.material as THREE.MeshBasicMaterial;
    mat.color.setHex(0xffcc66);
    mat.opacity = lightIntensity;
  });
};

// Generate a noise texture for water normals
function createWaterNormals(): THREE.Texture {
  const canvas = document.createElement('canvas');
  canvas.width = 512;
  canvas.height = 512;
  const context = canvas.getContext('2d');
  if (context) {
    context.fillStyle = '#8080ff'; // Default normal blue
    context.fillRect(0, 0, 512, 512);
    // Add some random noise
    for (let i = 0; i < 20000; i++) {
        const x = Math.random() * 512;
        const y = Math.random() * 512;
        const r = Math.random() * 255;
        const g = Math.random() * 255;
        // Simple noise approximation for ripples
        context.fillStyle = `rgb(${r}, ${g}, 255)`;
        context.fillRect(x, y, 2, 2);
    }
  }
  const texture = new THREE.CanvasTexture(canvas);
  texture.wrapS = THREE.RepeatWrapping;
  texture.wrapT = THREE.RepeatWrapping;
  return texture;
}

const init = (): void => {
  if (!canvasContainer.value) return;

  scene = new THREE.Scene();

  camera = new THREE.PerspectiveCamera(55, window.innerWidth / window.innerHeight, 1, 20000);
  camera.position.set(30, 30, 100);

  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 0.5;
  canvasContainer.value.appendChild(renderer.domElement);

  controls = new OrbitControls(camera, renderer.domElement);
  controls.maxPolarAngle = Math.PI * 0.495;
  controls.target.set(0, 10, 0);
  controls.minDistance = 40.0;
  controls.maxDistance = 2000.0;
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.update();

  sun = new THREE.Vector3();
  sky = new Sky();
  sky.scale.setScalar(10000);
  scene.add(sky);

  const skyUniforms = (sky.material as THREE.ShaderMaterial).uniforms;
  skyUniforms['turbidity']!.value = 10;
  skyUniforms['rayleigh']!.value = 2;
  skyUniforms['mieCoefficient']!.value = 0.005;
  skyUniforms['mieDirectionalG']!.value = 0.8;

  pmremGenerator = new THREE.PMREMGenerator(renderer);
  sceneEnv = new THREE.Scene();
  renderTarget = null;

  const waterGeometry = new THREE.PlaneGeometry(10000, 10000);
  water = new Water(
    waterGeometry,
    {
      textureWidth: 512,
      textureHeight: 512,
      waterNormals: createWaterNormals(),
      sunDirection: new THREE.Vector3(),
      sunColor: 0xffffff,
      waterColor: 0x001e0f,
      distortionScale: 3.7,
      fog: true
    }
  );
  water.rotation.x = -Math.PI / 2;
  scene.add(water);

  ambientLight = new THREE.AmbientLight(0xcccccc, 0.4);
  scene.add(ambientLight);

  sunLight = new THREE.DirectionalLight(0xffaa33, 1);
  sunLight.position.set(-1, 1, 1);
  scene.add(sunLight);

  scene.fog = new THREE.FogExp2(0xefd1b5, 0.0015);

  buildBridge();
  updateSceneForTime(timeOfDay.value);
  refreshEnvironmentMap();

  window.addEventListener('resize', onWindowResize);

  lastFrameTime = performance.now();
  animate();
};

const createBridgeLight = (x: number, y: number, z: number, group: THREE.Group): void => {
  const light = new THREE.PointLight(0xffcc66, 0, 15, 2);
  light.position.set(x, y, z);
  group.add(light);
  bridgeLights.push(light);

  const bulbGeo = new THREE.SphereGeometry(0.8, 8, 8);
  const bulbMat = new THREE.MeshBasicMaterial({ 
    color: 0xffcc66, 
    transparent: true,
    opacity: 0
  });
  const bulb = new THREE.Mesh(bulbGeo, bulbMat);
  bulb.position.set(x, y, z);
  group.add(bulb);
  bridgeLightMeshes.push(bulb);
};

const buildBridge = (): void => {
  const bridgeGroup = new THREE.Group();
  scene.add(bridgeGroup);

  const towerMat = new THREE.MeshStandardMaterial({
    color: BRIDGE_COLOR,
    roughness: 0.7,
    metalness: 0.1
  });

  const roadMat = new THREE.MeshStandardMaterial({
    color: ROAD_COLOR,
    roughness: 0.9
  });

  const cableMat = new THREE.MeshStandardMaterial({
    color: CABLE_COLOR,
    roughness: 0.5,
    metalness: 0.2
  });

  const towerHeight = 100;
  const towerWidth = 10;
  const towerDepth = 6;
  const span = 400;
  const sideSpan = 150;
  const deckY = 25;

  const createTower = (x: number): THREE.Group => {
    const towerGroup = new THREE.Group();
    towerGroup.position.set(x, 0, 0);

    const legGeo = new THREE.BoxGeometry(towerWidth, towerHeight, towerDepth);
    const legLeft = new THREE.Mesh(legGeo, towerMat);
    legLeft.position.set(0, towerHeight / 2, 15);
    legLeft.castShadow = true;
    legLeft.receiveShadow = true;

    const legRight = new THREE.Mesh(legGeo, towerMat);
    legRight.position.set(0, towerHeight / 2, -15);
    legRight.castShadow = true;
    legRight.receiveShadow = true;

    const braceGeo = new THREE.BoxGeometry(towerWidth - 2, 4, 30);
    const brace1 = new THREE.Mesh(braceGeo, towerMat);
    brace1.position.set(0, towerHeight * 0.9, 0);

    const brace2 = new THREE.Mesh(braceGeo, towerMat);
    brace2.position.set(0, towerHeight * 0.7, 0);

    const brace3 = new THREE.Mesh(braceGeo, towerMat);
    brace3.position.set(0, towerHeight * 0.5, 0);

    const brace4 = new THREE.Mesh(braceGeo, towerMat);
    brace4.position.set(0, deckY + 5, 0);

    const topGeo = new THREE.BoxGeometry(towerWidth - 2, 10, towerDepth - 2);
    const topLeft = new THREE.Mesh(topGeo, towerMat);
    topLeft.position.set(0, towerHeight + 5, 15);
    const topRight = new THREE.Mesh(topGeo, towerMat);
    topRight.position.set(0, towerHeight + 5, -15);

    towerGroup.add(legLeft, legRight, brace1, brace2, brace3, brace4, topLeft, topRight);

    createBridgeLight(0, towerHeight + 10, 15, towerGroup);
    createBridgeLight(0, towerHeight + 10, -15, towerGroup);
    createBridgeLight(0, towerHeight * 0.9 + 2, 0, towerGroup);
    createBridgeLight(0, towerHeight * 0.7 + 2, 0, towerGroup);

    return towerGroup;
  };

  const tower1 = createTower(-span / 2);
  const tower2 = createTower(span / 2);
  bridgeGroup.add(tower1, tower2);

  const totalLength = span + sideSpan * 2;
  const deckGeo = new THREE.BoxGeometry(totalLength, 2, 34);
  const deck = new THREE.Mesh(deckGeo, roadMat);
  deck.position.set(0, deckY, 0);
  deck.receiveShadow = true;
  bridgeGroup.add(deck);

  for (let dx = -totalLength / 2 + 20; dx <= totalLength / 2 - 20; dx += 40) {
    createBridgeLight(dx, deckY + 3, 12, bridgeGroup);
    createBridgeLight(dx, deckY + 3, -12, bridgeGroup);
  }

  const createMainCable = (zOffset: number): THREE.Vector3[] => {
    const curve1 = new THREE.QuadraticBezierCurve3(
      new THREE.Vector3(-span / 2 - sideSpan, deckY, zOffset),
      new THREE.Vector3(-span / 2 - sideSpan / 2, deckY + (towerHeight - deckY) / 2, zOffset),
      new THREE.Vector3(-span / 2, towerHeight, zOffset)
    );

    const curve2 = new THREE.QuadraticBezierCurve3(
      new THREE.Vector3(-span / 2, towerHeight, zOffset),
      new THREE.Vector3(0, deckY + 5, zOffset),
      new THREE.Vector3(span / 2, towerHeight, zOffset)
    );

    const curve3 = new THREE.QuadraticBezierCurve3(
      new THREE.Vector3(span / 2, towerHeight, zOffset),
      new THREE.Vector3(span / 2 + sideSpan / 2, deckY + (towerHeight - deckY) / 2, zOffset),
      new THREE.Vector3(span / 2 + sideSpan, deckY, zOffset)
    );

    const points: THREE.Vector3[] = [
      ...curve1.getPoints(20),
      ...curve2.getPoints(50),
      ...curve3.getPoints(20)
    ];

    const curvePath = new THREE.CatmullRomCurve3(points);
    const tubeGeo = new THREE.TubeGeometry(curvePath, 100, 1.5, 8, false);
    const cableMesh = new THREE.Mesh(tubeGeo, cableMat);
    bridgeGroup.add(cableMesh);

    return points;
  };

  const leftCablePoints = createMainCable(15);
  const rightCablePoints = createMainCable(-15);

  const suspenderCount = leftCablePoints.length + rightCablePoints.length;
  const suspenderGeo = new THREE.CylinderGeometry(0.3, 0.3, 1, 8);
  const suspenderMesh = new THREE.InstancedMesh(suspenderGeo, cableMat, suspenderCount);

  const dummy = new THREE.Object3D();
  let idx = 0;

  [leftCablePoints, rightCablePoints].forEach((points) => {
    points.forEach((p) => {
      if (p.y > deckY + 2) {
        const height = p.y - deckY;
        dummy.position.set(p.x, deckY + height / 2, p.z);
        dummy.scale.set(1, height, 1);
        dummy.updateMatrix();
        suspenderMesh.setMatrixAt(idx++, dummy.matrix);
      }
    });
  });

  bridgeGroup.add(suspenderMesh);
};

const animate = (): void => {
  animationId = requestAnimationFrame(animate);

  const now = performance.now();
  const delta = (now - lastFrameTime) / 1000;
  lastFrameTime = now;

  if (isPlaying.value) {
    timeOfDay.value = (timeOfDay.value + delta * speedMultiplier.value / 86400) % 1;
    updateSceneForTime(timeOfDay.value);
  }

  if (water) {
    (water.material as THREE.ShaderMaterial).uniforms['time']!.value += delta;
  }
  controls.update();
  renderer.render(scene, camera);
};

const togglePlay = (): void => {
  isPlaying.value = !isPlaying.value;
  if (!isPlaying.value) {
    updateSceneForTime(timeOfDay.value);
    refreshEnvironmentMap();
  }
};

const setSpeed = (speed: number): void => {
  speedMultiplier.value = speed;
};

const setPresetTime = (preset: number): void => {
  timeOfDay.value = preset;
  updateSceneForTime(timeOfDay.value);
  refreshEnvironmentMap();
};

const onSliderPointerDown = (): void => {
};

const onSliderPointerUp = (): void => {
  updateSceneForTime(timeOfDay.value);
  refreshEnvironmentMap();
};

const onTimeSliderInput = (e: Event): void => {
  const target = e.target as HTMLInputElement;
  timeOfDay.value = parseFloat(target.value);
  updateSceneForTime(timeOfDay.value);
};

onMounted(() => {
  init();
});

onBeforeUnmount(() => {
  cleanUp();
});
</script>

<template>
  <div ref="canvasContainer" class="canvas-container"></div>
  <div class="overlay">
    <h1>金门大桥</h1>
  </div>
  <div class="time-control-panel">
    <div class="time-display">{{ formattedTime }}</div>
    <div class="slider-row">
      <input
        type="range"
        min="0"
        max="1"
        step="0.0001"
        :value="timeOfDay"
        @input="onTimeSliderInput"
        @pointerdown="onSliderPointerDown"
        @pointerup="onSliderPointerUp"
        @pointercancel="onSliderPointerUp"
        class="time-slider"
      />
    </div>
    <div class="controls-row">
      <button @click="togglePlay" class="ctrl-btn play-btn">
        {{ isPlaying ? '⏸' : '▶' }}
      </button>
      <div class="speed-btns">
        <button
          @click="setSpeed(TIME_SPEED.NORMAL)"
          :class="['ctrl-btn', 'speed-btn', { active: speedMultiplier === TIME_SPEED.NORMAL }]"
        >1x</button>
        <button
          @click="setSpeed(TIME_SPEED.FAST)"
          :class="['ctrl-btn', 'speed-btn', { active: speedMultiplier === TIME_SPEED.FAST }]"
        >10x</button>
        <button
          @click="setSpeed(TIME_SPEED.SUPER_FAST)"
          :class="['ctrl-btn', 'speed-btn', { active: speedMultiplier === TIME_SPEED.SUPER_FAST }]"
        >100x</button>
      </div>
    </div>
    <div class="preset-btns">
      <button @click="setPresetTime(PRESET_TIMES.SUNRISE)" class="ctrl-btn preset-btn">日出</button>
      <button @click="setPresetTime(PRESET_TIMES.NOON)" class="ctrl-btn preset-btn">正午</button>
      <button @click="setPresetTime(PRESET_TIMES.SUNSET)" class="ctrl-btn preset-btn">日落</button>
      <button @click="setPresetTime(PRESET_TIMES.MIDNIGHT)" class="ctrl-btn preset-btn">午夜</button>
    </div>
  </div>
</template>

<style scoped>
.canvas-container {
  width: 100vw;
  height: 100vh;
  position: absolute;
  top: 0;
  left: 0;
  z-index: 1;
}

.overlay {
  position: absolute;
  bottom: 30px;
  left: 30px;
  z-index: 2;
  color: white;
  font-family: 'Helvetica Neue', Arial, sans-serif;
  text-shadow: 0 2px 4px rgba(0,0,0,0.8);
  pointer-events: none;
}

.overlay h1 {
  margin: 0;
  font-size: 2.5rem;
  letter-spacing: 2px;
  font-weight: 300;
}

.time-control-panel {
  position: absolute;
  bottom: 20px;
  right: 20px;
  z-index: 10;
  background: rgba(0, 0, 0, 0.6);
  backdrop-filter: blur(10px);
  border-radius: 12px;
  padding: 12px 16px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  min-width: 220px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  user-select: none;
}

.time-display {
  color: white;
  font-family: 'SF Mono', 'Consolas', monospace;
  font-size: 1.4rem;
  font-weight: 500;
  text-align: center;
  letter-spacing: 2px;
  text-shadow: 0 1px 2px rgba(0,0,0,0.5);
}

.slider-row {
  width: 100%;
}

.time-slider {
  width: 100%;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: linear-gradient(90deg, #1a1a3a 0%, #ff6b35 25%, #87ceeb 50%, #ff6b35 75%, #1a1a3a 100%);
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.time-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: white;
  cursor: pointer;
  box-shadow: 0 2px 6px rgba(0,0,0,0.4);
  transition: transform 0.15s ease;
}

.time-slider::-webkit-slider-thumb:hover {
  transform: scale(1.15);
}

.time-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: white;
  cursor: pointer;
  border: none;
  box-shadow: 0 2px 6px rgba(0,0,0,0.4);
}

.controls-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
}

.ctrl-btn {
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  color: white;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.85rem;
  transition: all 0.15s ease;
  font-family: inherit;
}

.ctrl-btn:hover {
  background: rgba(255, 255, 255, 0.2);
}

.play-btn {
  width: 36px;
  height: 36px;
  font-size: 1rem;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.speed-btns {
  display: flex;
  gap: 4px;
}

.speed-btn {
  padding: 6px 10px;
  font-size: 0.8rem;
}

.speed-btn.active {
  background: rgba(255, 170, 51, 0.3);
  border-color: #ffaa33;
  color: #ffcc66;
}

.preset-btns {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 4px;
}

.preset-btn {
  padding: 6px 4px;
  font-size: 0.75rem;
}

@media (max-width: 640px) {
  .time-control-panel {
    bottom: 10px;
    right: 10px;
    left: 10px;
    min-width: auto;
    padding: 10px 12px;
  }

  .overlay {
    bottom: auto;
    top: 20px;
    left: 20px;
  }

  .overlay h1 {
    font-size: 1.5rem;
  }
}
</style>
