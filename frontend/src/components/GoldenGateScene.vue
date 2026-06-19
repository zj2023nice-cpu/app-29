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
const LIGHT_BULB_COLOR = 0xFFDD88;

// Time Control Constants
const SF_LATITUDE = 37.7749 * Math.PI / 180;
const MAX_SOLAR_DECLINATION = 23.44 * Math.PI / 180;
const DAY_HOURS = 24;
const SPEED_OPTIONS = [1, 10, 100] as const;
const PRESET_TIMES = { sunrise: 6, noon: 12, sunset: 18, midnight: 0 } as const;
const BULB_COUNT = 80;
const LIGHT_UPDATE_THRESHOLD = 0.02;
const LIGHT_ON_ALTITUDE = -0.15;
const LIGHT_OFF_ALTITUDE = 0.05;

// Time Control State
const timeOfDay = ref<number>(15);
const isPlaying = ref<boolean>(false);
const playSpeed = ref<number>(1);
const formattedTime = computed<string>(() => {
  const h = Math.floor(timeOfDay.value);
  const m = Math.floor((timeOfDay.value - h) * 60);
  return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}`;
});

let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let water: Water;
let sun: THREE.Vector3;
let sky: Sky;
let skyMaterial: THREE.ShaderMaterial;
let ambientLight: THREE.AmbientLight;
let dirLight: THREE.DirectionalLight;
let bridgeLights: THREE.PointLight[] = [];
let bulbMesh: THREE.InstancedMesh;
let bulbMaterial: THREE.MeshStandardMaterial;
let fog: THREE.FogExp2;
let pmremGenerator: THREE.PMREMGenerator;
let renderTarget: THREE.WebGLRenderTarget;
let sceneEnv: THREE.Scene;
let animationId: number;
let lastEnvUpdateTime = -1;

const COL_DIR_DAY = new THREE.Color(0xfff5e0);
const COL_DIR_SUNSET = new THREE.Color(0xff8833);
const COL_DIR_NIGHT = new THREE.Color(0x334466);
const COL_AMB_DAY = new THREE.Color(0x99aabb);
const COL_AMB_NIGHT = new THREE.Color(0x0a1020);
const COL_AMB_SUNSET = new THREE.Color(0xcc6633);
const COL_FOG_DAY = new THREE.Color(0xc8d8e8);
const COL_FOG_NIGHT = new THREE.Color(0x0a1525);
const COL_FOG_SUNSET = new THREE.Color(0xdd8855);
const COL_TMP = new THREE.Color();
const VEC_SUN_TMP = new THREE.Vector3();

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

const getDayOfYear = (date: Date): number => {
  const start = new Date(date.getFullYear(), 0, 0);
  const diff = date.getTime() - start.getTime();
  const oneDay = 1000 * 60 * 60 * 24;
  return Math.floor(diff / oneDay);
};

const getSolarDeclination = (date: Date): number => {
  const dayOfYear = getDayOfYear(date);
  const gamma = (2 * Math.PI / 365) * (dayOfYear - 1);
  const declination = MAX_SOLAR_DECLINATION * Math.sin(gamma - 1.3944 + 0.0334 * Math.sin(gamma - 0.0489));
  return declination;
};

const calculateSunPosition = (hours: number, date: Date = new Date()): { elevation: number; azimuth: number; altitude: number; declination: number } => {
  const solarNoon = 12;
  const hourAngle = (hours - solarNoon) * 15 * Math.PI / 180;
  const lat = SF_LATITUDE;
  const dec = getSolarDeclination(date);
  const sinAlt = Math.sin(lat) * Math.sin(dec) + Math.cos(lat) * Math.cos(dec) * Math.cos(hourAngle);
  const altitude = Math.asin(Math.max(-1, Math.min(1, sinAlt)));
  const cosAz = (Math.sin(dec) - Math.sin(altitude) * Math.sin(lat)) / (Math.cos(altitude) * Math.cos(lat) + 0.0001);
  let azimuth = Math.acos(Math.max(-1, Math.min(1, cosAz)));
  if (hourAngle > 0) azimuth = 2 * Math.PI - azimuth;
  const elevation = altitude * 180 / Math.PI;
  return { elevation, azimuth, altitude, declination: dec * 180 / Math.PI };
};

const smoothstep = (edge0: number, edge1: number, x: number): number => {
  const t = Math.max(0, Math.min(1, (x - edge0) / (edge1 - edge0)));
  return t * t * (3 - 2 * t);
};

const updateTimeOfDay = (hours: number): void => {
  const h = ((hours % DAY_HOURS) + DAY_HOURS) % DAY_HOURS;
  timeOfDay.value = h;
  const sunData = calculateSunPosition(h);
  const altitude = sunData.altitude;
  const azimuth = sunData.azimuth;
  VEC_SUN_TMP.set(
    Math.cos(altitude) * Math.sin(azimuth),
    Math.sin(altitude),
    Math.cos(altitude) * Math.cos(azimuth)
  );
  sun.copy(VEC_SUN_TMP);
  if (skyMaterial) {
    skyMaterial.uniforms['sunPosition']!.value.copy(sun);
    const turbidity = 8 + Math.max(0, -altitude) * 20;
    const rayleigh = altitude > 0.1 ? 2 + (1 - Math.min(1, altitude / (Math.PI / 4))) * 3 : 4;
    skyMaterial.uniforms['turbidity']!.value = turbidity;
    skyMaterial.uniforms['rayleigh']!.value = rayleigh;
  }
  const waterMat = water?.material as THREE.ShaderMaterial;
  if (waterMat) {
    waterMat.uniforms['sunDirection']!.value.copy(sun).normalize();
  }
  if (dirLight) {
    dirLight.position.copy(sun).multiplyScalar(100);
    const dayIntensity = smoothstep(-0.1, 0.15, altitude);
    if (altitude > 0) {
      COL_TMP.lerpColors(COL_DIR_SUNSET, COL_DIR_DAY, smoothstep(0, 0.3, altitude));
      dirLight.color.copy(COL_TMP);
      dirLight.intensity = dayIntensity * 1.2;
    } else {
      dirLight.color.copy(COL_DIR_NIGHT);
      dirLight.intensity = 0.1;
    }
  }
  if (ambientLight) {
    const dayFactor = smoothstep(-0.15, 0.2, altitude);
    if (altitude > 0) {
      COL_TMP.lerpColors(COL_AMB_SUNSET, COL_AMB_DAY, smoothstep(0, 0.3, altitude));
    } else {
      COL_TMP.lerpColors(COL_AMB_NIGHT, COL_AMB_SUNSET, smoothstep(-0.3, 0, altitude));
    }
    ambientLight.color.copy(COL_TMP);
    ambientLight.intensity = 0.15 + dayFactor * 0.45;
  }
  if (fog) {
    const dayFactor = smoothstep(-0.15, 0.2, altitude);
    if (altitude > 0) {
      COL_TMP.lerpColors(COL_FOG_SUNSET, COL_FOG_DAY, smoothstep(0, 0.3, altitude));
    } else {
      COL_TMP.lerpColors(COL_FOG_NIGHT, COL_FOG_SUNSET, smoothstep(-0.3, 0, altitude));
    }
    fog.color.copy(COL_TMP);
    fog.density = 0.0008 + (1 - dayFactor) * 0.002;
  }
  renderer.toneMappingExposure = 0.3 + smoothstep(-0.2, 0.3, altitude) * 0.5;
  const lightFactor = 1 - smoothstep(LIGHT_ON_ALTITUDE, LIGHT_OFF_ALTITUDE, altitude);
  const lightsOn = lightFactor > 0.01;
  if (bulbMesh) {
    bulbMesh.visible = lightsOn;
  }
  if (bulbMaterial && lightsOn) {
    bulbMaterial.emissive.setHex(LIGHT_BULB_COLOR);
    bulbMaterial.emissiveIntensity = 0.5 + lightFactor * 1.5;
  }
  bridgeLights.forEach(light => {
    light.intensity = lightsOn ? lightFactor * 1.5 : 0;
  });
  if (renderer && Math.abs(h - lastEnvUpdateTime) > LIGHT_UPDATE_THRESHOLD) {
    lastEnvUpdateTime = h;
    if (renderTarget) renderTarget.dispose();
    if (sky && sceneEnv) {
      sceneEnv.add(sky);
      renderTarget = pmremGenerator.fromScene(sceneEnv);
      scene.add(sky);
      scene.environment = renderTarget.texture;
    }
  }
};

const togglePlay = (): void => {
  isPlaying.value = !isPlaying.value;
};

const setSpeed = (speed: number): void => {
  playSpeed.value = speed;
};

const setPreset = (preset: keyof typeof PRESET_TIMES): void => {
  timeOfDay.value = PRESET_TIMES[preset];
  updateTimeOfDay(timeOfDay.value);
};

const onTimeSlider = (e: Event): void => {
  const target = e.target as HTMLInputElement;
  const val = parseFloat(target.value);
  updateTimeOfDay(val);
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

  // 1. Scene Setup
  scene = new THREE.Scene();
  
  // 2. Camera
  camera = new THREE.PerspectiveCamera(55, window.innerWidth / window.innerHeight, 1, 20000);
  camera.position.set(30, 30, 100);

  // 3. Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 0.5;
  canvasContainer.value.appendChild(renderer.domElement);

  // 4. Controls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.maxPolarAngle = Math.PI * 0.495;
  controls.target.set(0, 10, 0);
  controls.minDistance = 40.0;
  controls.maxDistance = 2000.0;
  controls.update();

  // 5. Sun & Sky
  sun = new THREE.Vector3();
  sky = new Sky();
  sky.scale.setScalar(10000);
  scene.add(sky);
  skyMaterial = sky.material as THREE.ShaderMaterial;
  skyMaterial.uniforms['turbidity']!.value = 10;
  skyMaterial.uniforms['rayleigh']!.value = 2;
  skyMaterial.uniforms['mieCoefficient']!.value = 0.005;
  skyMaterial.uniforms['mieDirectionalG']!.value = 0.8;

  pmremGenerator = new THREE.PMREMGenerator(renderer);
  sceneEnv = new THREE.Scene();
  renderTarget = undefined as unknown as THREE.WebGLRenderTarget;

  // 6. Water
  const waterGeometry = new THREE.PlaneGeometry(10000, 10000);
  water = new Water(
    waterGeometry,
    {
      textureWidth: 512,
      textureHeight: 512,
      waterNormals: createWaterNormals(),
      sunDirection: new THREE.Vector3(),
      sunColor: 0xffffff,
      waterColor: 0x001e2f,
      distortionScale: 3.7,
      fog: scene.fog !== undefined
    }
  );
  water.rotation.x = -Math.PI / 2;
  scene.add(water);

  // 7. Lighting (Atmospheric)
  ambientLight = new THREE.AmbientLight(0xcccccc, 0.4);
  scene.add(ambientLight);

  dirLight = new THREE.DirectionalLight(0xffaa33, 1);
  dirLight.position.set(-1, 1, 1);
  scene.add(dirLight);

  // Fog for depth
  fog = new THREE.FogExp2(0xc8d8e8, 0.001);
  scene.fog = fog;

  // 8. Build The Bridge
  buildBridge();

  // Event Listeners
  window.addEventListener('resize', onWindowResize);

  // Initialize time
  updateTimeOfDay(timeOfDay.value);

  // Start Loop
  animate();
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

  // --- Dimensions (Approximate Scaled) ---
  const towerHeight = 100; // Above water
  const towerWidth = 10;
  const towerDepth = 6;
  const span = 400; // Distance between towers
  const sideSpan = 150;
  const deckY = 25; // Height of deck above water

  // --- Helper: Create Tower ---
  const createTower = (x: number): THREE.Group => {
    const towerGroup = new THREE.Group();
    towerGroup.position.set(x, 0, 0);

    // Two legs
    const legGeo = new THREE.BoxGeometry(towerWidth, towerHeight, towerDepth);
    const legLeft = new THREE.Mesh(legGeo, towerMat);
    legLeft.position.set(0, towerHeight / 2, 15);
    legLeft.castShadow = true;
    legLeft.receiveShadow = true;

    const legRight = new THREE.Mesh(legGeo, towerMat);
    legRight.position.set(0, towerHeight / 2, -15);
    legRight.castShadow = true;
    legRight.receiveShadow = true;

    // Cross braces (Art Deco style)
    const braceGeo = new THREE.BoxGeometry(towerWidth - 2, 4, 30);
    const brace1 = new THREE.Mesh(braceGeo, towerMat);
    brace1.position.set(0, towerHeight * 0.9, 0);
    
    const brace2 = new THREE.Mesh(braceGeo, towerMat);
    brace2.position.set(0, towerHeight * 0.7, 0);

    const brace3 = new THREE.Mesh(braceGeo, towerMat);
    brace3.position.set(0, towerHeight * 0.5, 0);

    const brace4 = new THREE.Mesh(braceGeo, towerMat);
    brace4.position.set(0, deckY + 5, 0); // Below deck

    // Decorative top
    const topGeo = new THREE.BoxGeometry(towerWidth - 2, 10, towerDepth - 2);
    const topLeft = new THREE.Mesh(topGeo, towerMat);
    topLeft.position.set(0, towerHeight + 5, 15);
    const topRight = new THREE.Mesh(topGeo, towerMat);
    topRight.position.set(0, towerHeight + 5, -15);

    towerGroup.add(legLeft, legRight, brace1, brace2, brace3, brace4, topLeft, topRight);
    return towerGroup;
  };

  const tower1 = createTower(-span / 2);
  const tower2 = createTower(span / 2);
  bridgeGroup.add(tower1, tower2);

  // --- Deck ---
  const totalLength = span + (sideSpan * 2);
  const deckGeo = new THREE.BoxGeometry(totalLength, 2, 34); // Wide deck
  const deck = new THREE.Mesh(deckGeo, roadMat);
  deck.position.set(0, deckY, 0);
  deck.receiveShadow = true;
  bridgeGroup.add(deck);

  // --- Main Cables (Catenary) ---
  // Using quadratic curves to simulate catenary
  const createMainCable = (zOffset: number): THREE.Vector3[] => {
    // Left Side Span
    const curve1 = new THREE.QuadraticBezierCurve3(
      new THREE.Vector3(-span/2 - sideSpan, deckY, zOffset),
      new THREE.Vector3(-span/2 - sideSpan/2, deckY + (towerHeight - deckY)/2, zOffset), // Control point
      new THREE.Vector3(-span/2, towerHeight, zOffset)
    );

    // Center Span (The deep curve)
    const curve2 = new THREE.QuadraticBezierCurve3(
      new THREE.Vector3(-span/2, towerHeight, zOffset),
      new THREE.Vector3(0, deckY + 5, zOffset), // Low point
      new THREE.Vector3(span/2, towerHeight, zOffset)
    );

    // Right Side Span
    const curve3 = new THREE.QuadraticBezierCurve3(
      new THREE.Vector3(span/2, towerHeight, zOffset),
      new THREE.Vector3(span/2 + sideSpan/2, deckY + (towerHeight - deckY)/2, zOffset),
      new THREE.Vector3(span/2 + sideSpan, deckY, zOffset)
    );

    const points = [
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

  // --- Vertical Suspenders (InstancedMesh for performance) ---
  // We place a vertical line from the cable point down to the deck
  // Only for the middle span mostly, and parts of side spans
  const suspenderCount = leftCablePoints.length + rightCablePoints.length;
  const suspenderGeo = new THREE.CylinderGeometry(0.3, 0.3, 1, 8);
  const suspenderMesh = new THREE.InstancedMesh(suspenderGeo, cableMat, suspenderCount);
  
  const dummy = new THREE.Object3D();
  let idx = 0;

  [leftCablePoints, rightCablePoints].forEach(points => {
    points.forEach((p) => {
        // Only add suspenders if the point is above the deck significantly
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

  // Bridge Lights
  const bulbGeo = new THREE.SphereGeometry(0.8, 6, 6);
  bulbMaterial = new THREE.MeshStandardMaterial({
    color: 0x111111,
    emissive: LIGHT_BULB_COLOR,
    emissiveIntensity: 0,
    roughness: 0.3,
    metalness: 0
  });
  bulbMesh = new THREE.InstancedMesh(bulbGeo, bulbMaterial, BULB_COUNT);
  bulbMesh.visible = false;
  const bulbDummy = new THREE.Object3D();
  let bulbIdx = 0;
  const lightPositions: THREE.Vector3[] = [];

  const towerPositions = [-span / 2, span / 2];
  towerPositions.forEach(tx => {
    for (let z = -15; z <= 15; z += 30) {
      for (let y = deckY + 10; y <= towerHeight - 5; y += 20) {
        bulbDummy.position.set(tx, y, z);
        bulbDummy.updateMatrix();
        bulbMesh.setMatrixAt(bulbIdx++, bulbDummy.matrix);
        lightPositions.push(new THREE.Vector3(tx, y, z));
      }
      const topPos = new THREE.Vector3(tx, towerHeight + 2, z);
      bulbDummy.position.copy(topPos);
      bulbDummy.updateMatrix();
      bulbMesh.setMatrixAt(bulbIdx++, bulbDummy.matrix);
      lightPositions.push(topPos);
    }
  });

  const deckBulbCount = 40;
  for (let i = 0; i < deckBulbCount && bulbIdx < BULB_COUNT; i++) {
    const t = (i / deckBulbCount - 0.5) * totalLength;
    for (let z = -15; z <= 15; z += 30) {
      if (bulbIdx >= BULB_COUNT) break;
      const pos = new THREE.Vector3(t, deckY + 3, z);
      bulbDummy.position.copy(pos);
      bulbDummy.updateMatrix();
      bulbMesh.setMatrixAt(bulbIdx++, bulbDummy.matrix);
      lightPositions.push(pos);
    }
  }

  bulbMesh.count = bulbIdx;
  bulbMesh.instanceMatrix.needsUpdate = true;
  bridgeGroup.add(bulbMesh);

  const pointLightCount = Math.min(12, lightPositions.length);
  const step = Math.max(1, Math.floor(lightPositions.length / pointLightCount));
  for (let i = 0; i < lightPositions.length; i += step) {
    if (bridgeLights.length >= pointLightCount) break;
    const p = lightPositions[i];
    if (!p) continue;
    const pl = new THREE.PointLight(LIGHT_BULB_COLOR, 0, 15, 2);
    pl.position.copy(p);
    scene.add(pl);
    bridgeLights.push(pl);
  }
};

let lastFrameTime = performance.now();
const animate = (): void => {
  animationId = requestAnimationFrame(animate);
  const now = performance.now();
  const delta = (now - lastFrameTime) / 1000;
  lastFrameTime = now;

  if (isPlaying.value) {
    let newTime = timeOfDay.value + delta * playSpeed.value;
    if (newTime >= DAY_HOURS) newTime -= DAY_HOURS;
    updateTimeOfDay(newTime);
  }

  if (water) {
    (water.material as THREE.ShaderMaterial).uniforms['time']!.value += delta;
  }
  controls.update();
  renderer.render(scene, camera);
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
  <div class="time-control">
    <div class="time-display">{{ formattedTime }}</div>
    <div class="control-row">
      <button class="ctrl-btn play-btn" @click="togglePlay">
        {{ isPlaying ? '⏸' : '▶' }}
      </button>
      <input
        type="range"
        class="time-slider"
        min="0"
        max="24"
        step="0.01"
        :value="timeOfDay"
        @input="onTimeSlider"
      />
    </div>
    <div class="control-row">
      <div class="speed-group">
        <button
          v-for="sp in SPEED_OPTIONS"
          :key="sp"
          class="ctrl-btn speed-btn"
          :class="{ active: playSpeed === sp }"
          @click="setSpeed(sp)"
        >
          {{ sp }}x
        </button>
      </div>
      <div class="preset-group">
        <button class="ctrl-btn preset-btn" @click="setPreset('sunrise')">日出</button>
        <button class="ctrl-btn preset-btn" @click="setPreset('noon')">正午</button>
        <button class="ctrl-btn preset-btn" @click="setPreset('sunset')">日落</button>
        <button class="ctrl-btn preset-btn" @click="setPreset('midnight')">午夜</button>
      </div>
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

.time-control {
  position: absolute;
  bottom: 20px;
  right: 20px;
  z-index: 10;
  background: rgba(0, 0, 0, 0.6);
  backdrop-filter: blur(8px);
  border-radius: 12px;
  padding: 12px 16px;
  min-width: 280px;
  color: white;
  font-family: 'Helvetica Neue', Arial, sans-serif;
  user-select: none;
}

.time-display {
  font-size: 1.4rem;
  font-weight: 500;
  text-align: center;
  margin-bottom: 8px;
  letter-spacing: 2px;
  font-variant-numeric: tabular-nums;
}

.control-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

.control-row:last-child {
  margin-bottom: 0;
}

.ctrl-btn {
  background: rgba(255, 255, 255, 0.15);
  border: 1px solid rgba(255, 255, 255, 0.25);
  color: white;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.8rem;
  padding: 4px 10px;
  transition: all 0.15s ease;
  font-family: inherit;
}

.ctrl-btn:hover {
  background: rgba(255, 255, 255, 0.25);
}

.ctrl-btn.active {
  background: rgba(255, 170, 51, 0.5);
  border-color: rgba(255, 170, 51, 0.8);
}

.play-btn {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1rem;
  padding: 0;
  flex-shrink: 0;
}

.time-slider {
  flex: 1;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: linear-gradient(to right, #1a1a3e, #ff8833, #ffdd88, #ff8833, #1a1a3e);
  outline: none;
  cursor: pointer;
}

.time-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: white;
  cursor: pointer;
  box-shadow: 0 1px 4px rgba(0,0,0,0.4);
}

.time-slider::-moz-range-thumb {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: white;
  cursor: pointer;
  border: none;
  box-shadow: 0 1px 4px rgba(0,0,0,0.4);
}

.speed-group {
  display: flex;
  gap: 4px;
}

.preset-group {
  display: flex;
  gap: 4px;
  margin-left: auto;
}

.preset-btn {
  font-size: 0.75rem;
  padding: 4px 8px;
}

@media (max-width: 600px) {
  .time-control {
    right: 10px;
    bottom: 10px;
    left: 10px;
    min-width: auto;
    padding: 10px 12px;
  }
  .preset-btn {
    font-size: 0.7rem;
    padding: 4px 6px;
  }
  .speed-btn {
    font-size: 0.75rem;
    padding: 4px 8px;
  }
}
</style>
