<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref, computed } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { Water } from 'three/examples/jsm/objects/Water.js';
import { Sky } from 'three/examples/jsm/objects/Sky.js';

const canvasContainer = ref<HTMLDivElement | null>(null);

// --- Configuration Constants ---
const BRIDGE_COLOR = 0xF04A00;
const ROAD_COLOR = 0x333333;
const CABLE_COLOR = 0xF04A00;
const LIGHT_COLOR = 0xFFCC66;

// San Francisco geographic constants (radians)
const SF_LATITUDE = 37.7749 * Math.PI / 180;
const SOLAR_DECLINATION = 23.44 * Math.PI / 180; // Summer solstice for long daylight
const MINUTES_PER_DAY = 1440;
const DEG_PER_HOUR = 15 * Math.PI / 180;

// Time preset hours
const PRESET_SUNRISE = 5.83;   // ~05:50
const PRESET_NOON = 12;
const PRESET_SUNSET = 20.5;    // ~20:30
const PRESET_MIDNIGHT = 0;
const SPEED_LEVELS = [1, 10, 100] as const;
// PMREM rebuild rate limit: at most one rebuild every N seconds, regardless of speed
const PMREM_MIN_INTERVAL = 0.5;

// --- Time Control State (reactive for UI) ---
const timeMinutes = ref<number>(12 * 60);
const isPlaying = ref<boolean>(false);
const speedIndex = ref<number>(0);
const isDragging = ref<boolean>(false);
const timeNeedsForceUpdate = ref<boolean>(false);

// Normalize hours to [0, 24) so that 1440 min (= 24:00) renders identical to 00:00
const timeOfDay = computed<number>(() => (timeMinutes.value % MINUTES_PER_DAY) / 60);
const currentSpeed = computed<number>(() => SPEED_LEVELS[speedIndex.value] ?? 1);
const timeDisplay = computed<string>(() => {
  const total = timeMinutes.value;
  if (total >= MINUTES_PER_DAY) return '24:00';
  const h = Math.floor(total / 60);
  const m = Math.floor(total % 60);
  return `${String(h).padStart(2, '0')}:${String(m).padStart(2, '0')}`;
});

// --- Three.js Globals ---
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let water: Water;
let sun: THREE.Vector3;
let animationId: number;
let skyMesh: Sky;
let skyUniforms: Record<string, THREE.IUniform>;
let pmremGenerator: THREE.PMREMGenerator;
let renderTarget: THREE.WebGLRenderTarget | null = null;
let sceneEnv: THREE.Scene;
let sunDirectional: THREE.DirectionalLight;
let ambientLight: THREE.AmbientLight;
let waterUniforms: Record<string, THREE.IUniform>;
let fogExp: THREE.FogExp2;

// Shared material for all bridge bulbs (one instance, updated once per frame)
let bulbMaterial: THREE.MeshStandardMaterial;
// Point lights attached to key bulbs (small fixed count for performance)
const bridgePointLights: THREE.PointLight[] = [];

// Preallocated color scratch objects (reused every frame to avoid GC)
const colWater = new THREE.Color();
const colSun = new THREE.Color();
const colDirLight = new THREE.Color();
const colAmbient = new THREE.Color();
const colFog = new THREE.Color();
const colDay = new THREE.Color(0xaabbdd);
const colNightAmb = new THREE.Color(0x1a2040);
const colWaterDay = new THREE.Color(0x0a3a5c);
const colWaterSunset = new THREE.Color(0x2a1810);
const colWaterNight = new THREE.Color(0x020812);
const colSunWhite = new THREE.Color(0xffffff);
const colSunGolden = new THREE.Color(0xffaa44);
const colSunOrange = new THREE.Color(0xff6622);
const colFogDay = new THREE.Color(0xbfd9ee);
const colFogSunset = new THREE.Color(0xd09060);
const colFogNight = new THREE.Color(0x050812);

// Track last PMREM rebuild wall-clock time for rate limiting
let lastPMREMTime = -Infinity;
let lastClockTime = 0;

// --- Sun Position (San Francisco, solar geometry) ---
interface SunPosition {
  altitude: number;
  azimuth: number;
  direction: THREE.Vector3;
}

function computeSunPosition(hours: number): SunPosition {
  const hra = (hours - PRESET_NOON) * DEG_PER_HOUR;
  const sinLat = Math.sin(SF_LATITUDE);
  const cosLat = Math.cos(SF_LATITUDE);
  const sinDec = Math.sin(SOLAR_DECLINATION);
  const cosDec = Math.cos(SOLAR_DECLINATION);
  const cosHra = Math.cos(hra);

  const sinAlt = sinLat * sinDec + cosLat * cosDec * cosHra;
  const altitude = Math.asin(Math.max(-1, Math.min(1, sinAlt)));

  const cosAlt = Math.cos(altitude);
  let azimuth: number;
  if (cosAlt < 1e-6) {
    azimuth = 0;
  } else {
    const cosAz = (sinDec - sinAlt * sinLat) / (cosAlt * cosLat);
    const cosAzClamped = Math.max(-1, Math.min(1, cosAz));
    if (hours < PRESET_NOON) {
      azimuth = Math.PI * 2 - Math.acos(cosAzClamped);
    } else {
      azimuth = Math.acos(cosAzClamped);
    }
  }

  // azimuth 0=north(+Z), PI/2=east(-X), PI=south(-Z), 3PI/2=west(+X)
  const cosAlt2 = Math.cos(altitude);
  const dx = -cosAlt2 * Math.sin(azimuth);
  const dy = Math.sin(altitude);
  const dz = -cosAlt2 * Math.cos(azimuth);
  const direction = new THREE.Vector3(dx, dy, dz).normalize();

  return { altitude, azimuth, direction };
}

// --- Interpolation utilities ---
function smoothstep(edge0: number, edge1: number, x: number): number {
  const t = Math.max(0, Math.min(1, (x - edge0) / (edge1 - edge0)));
  return t * t * (3 - 2 * t);
}

function lerp3(out: THREE.Color, a: THREE.Color, b: THREE.Color, t: number): THREE.Color {
  out.copy(a).lerp(b, t);
  return out;
}

// --- Rebuild PMREM environment map ---
function rebuildEnvMap(): void {
  if (renderTarget) renderTarget.dispose();
  sceneEnv.add(skyMesh);
  renderTarget = pmremGenerator.fromScene(sceneEnv);
  scene.add(skyMesh);
  scene.environment = renderTarget.texture;
}

// --- Drive all visual parameters from hours-of-day ---
function updateSceneForTime(hours: number, now: number, forceEnv: boolean = false): void {
  const sp = computeSunPosition(hours);
  const alt = sp.altitude;
  const dir = sp.direction;

  sun.copy(dir);
  skyUniforms['sunPosition']!.value.copy(sun);
  waterUniforms['sunDirection']!.value.copy(dir);

  // Day factor: 1 = full day, 0 = full night
  const dayFactor = smoothstep(-0.04, 0.09, alt); // ~-2.3° to 5.2°

  // Sunset/golden-hour strength: peaks when sun near horizon
  // Gaussian-like bell centered at alt=0.12 rad (~7°), falls off by alt=0.4
  const sunsetBell = Math.exp(-Math.pow((alt - 0.12) / 0.18, 2));
  const sunsetK = sunsetBell * smoothstep(-0.05, 0.05, alt);

  // --- Sky ---
  const turbidity = 2 + 8 * (1 - dayFactor) + 2 * (1 - Math.min(1, Math.abs(alt) / 1.0));
  skyUniforms['turbidity']!.value = Math.max(2, Math.min(20, turbidity));

  const rayleigh = 0.5 + 3.5 * sunsetK;
  skyUniforms['rayleigh']!.value = Math.max(0.5, Math.min(4, rayleigh));

  const mie = 0.003 + 0.02 * (1 - dayFactor);
  skyUniforms['mieCoefficient']!.value = Math.max(0.003, Math.min(0.025, mie));
  skyUniforms['mieDirectionalG']!.value = 0.8;

  // Renderer exposure
  renderer.toneMappingExposure = 0.12 + 0.58 * dayFactor;

  // --- Water color ---
  lerp3(colWater, colWaterDay, colWaterSunset, sunsetK * 0.7);
  lerp3(colWater, colWaterNight, colWater, dayFactor);
  waterUniforms['waterColor']!.value.copy(colWater);

  // Sun specular color on water
  const sunK = smoothstep(0.3, 0.05, alt) * dayFactor;
  lerp3(colSun, colSunWhite, colSunGolden, sunK);
  lerp3(colSun, colSun, colSunOrange, sunsetK * 0.5);
  waterUniforms['sunColor']!.value.copy(colSun);

  // --- Sun directional light ---
  sunDirectional.intensity = dayFactor * 1.3;
  sunDirectional.position.copy(dir).multiplyScalar(500);
  lerp3(colDirLight, colSunWhite, colSunOrange, sunsetK * 0.8);
  sunDirectional.color.copy(colDirLight);

  // --- Ambient light ---
  ambientLight.intensity = 0.04 + 0.4 * dayFactor;
  lerp3(colAmbient, colNightAmb, colDay, dayFactor);
  ambientLight.color.copy(colAmbient);

  // --- Fog ---
  lerp3(colFog, colFogNight, colFogSunset, sunsetK * 0.6);
  lerp3(colFog, colFog, colFogDay, dayFactor);
  fogExp.color.copy(colFog);
  const fogDensity = 0.0007 + 0.0012 * sunsetK + 0.0004 * (1 - dayFactor);
  fogExp.density = Math.max(0.0004, Math.min(0.0025, fogDensity));

  // --- Bridge lights (shared material — one uniform update per frame) ---
  const lightIntensityFactor = smoothstep(0.0, -0.12, alt);
  if (bulbMaterial) {
    bulbMaterial.emissiveIntensity = 1.5 * lightIntensityFactor;
  }
  const plCount = bridgePointLights.length;
  for (let i = 0; i < plCount; i++) {
    bridgePointLights[i]!.intensity = 2.0 * lightIntensityFactor;
  }

  // --- PMREM rebuild: skip while dragging; rate-limit to at most once per PMREM_MIN_INTERVAL ---
  if (!isDragging.value && (forceEnv || now - lastPMREMTime >= PMREM_MIN_INTERVAL)) {
    rebuildEnvMap();
    lastPMREMTime = now;
  }
}

// --- Cleanup ---
const cleanUp = (): void => {
  if (animationId) cancelAnimationFrame(animationId);
  if (renderTarget) renderTarget.dispose();
  if (pmremGenerator) pmremGenerator.dispose();
  if (renderer) renderer.dispose();
  if (controls) controls.dispose();
  window.removeEventListener('resize', onWindowResize);
};

const onWindowResize = (): void => {
  if (!camera || !renderer) return;
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
};

// --- Water normals ---
function createWaterNormals(): THREE.Texture {
  const canvas = document.createElement('canvas');
  canvas.width = 512;
  canvas.height = 512;
  const context = canvas.getContext('2d');
  if (context) {
    context.fillStyle = '#8080ff';
    context.fillRect(0, 0, 512, 512);
    for (let i = 0; i < 20000; i++) {
      const x = Math.random() * 512;
      const y = Math.random() * 512;
      const r = Math.random() * 255;
      const g = Math.random() * 255;
      context.fillStyle = `rgb(${r}, ${g}, 255)`;
      context.fillRect(x, y, 2, 2);
    }
  }
  const texture = new THREE.CanvasTexture(canvas);
  texture.wrapS = THREE.RepeatWrapping;
  texture.wrapT = THREE.RepeatWrapping;
  return texture;
}

// --- Build bridge (augmented with lights) ---
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

  const bulbGeo = new THREE.SphereGeometry(0.8, 8, 6);
  const bulbColor = new THREE.Color(LIGHT_COLOR);
  // Single shared material for every bulb mesh
  bulbMaterial = new THREE.MeshStandardMaterial({
    color: 0x222222,
    emissive: bulbColor,
    emissiveIntensity: 0,
    roughness: 0.3
  });

  const towerHeight = 100;
  const towerWidth = 10;
  const towerDepth = 6;
  const span = 400;
  const sideSpan = 150;
  const deckY = 25;

  const makeBulb = (parent: THREE.Group, pos: THREE.Vector3, addPoint: boolean): void => {
    const bulb = new THREE.Mesh(bulbGeo, bulbMaterial);
    bulb.position.copy(pos);
    parent.add(bulb);
    if (addPoint) {
      const pl = new THREE.PointLight(bulbColor, 0, 25, 2);
      pl.position.copy(pos);
      parent.add(pl);
      bridgePointLights.push(pl);
    }
  };

  const createTower = (x: number) => {
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

    // Tower lights
    makeBulb(towerGroup, new THREE.Vector3(0, towerHeight + 10, 15), true);
    makeBulb(towerGroup, new THREE.Vector3(0, towerHeight + 10, -15), true);
    makeBulb(towerGroup, new THREE.Vector3(0, towerHeight * 0.9 - 2, 15), false);
    makeBulb(towerGroup, new THREE.Vector3(0, towerHeight * 0.9 - 2, -15), false);

    return towerGroup;
  };

  const tower1 = createTower(-span / 2);
  const tower2 = createTower(span / 2);
  bridgeGroup.add(tower1, tower2);

  const totalLength = span + (sideSpan * 2);
  const deckGeo = new THREE.BoxGeometry(totalLength, 2, 34);
  const deck = new THREE.Mesh(deckGeo, roadMat);
  deck.position.set(0, deckY, 0);
  deck.receiveShadow = true;
  bridgeGroup.add(deck);

  const createMainCable = (zOffset: number) => {
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
    const points = [
      ...curve1.getPoints(20),
      ...curve2.getPoints(50),
      ...curve3.getPoints(20)
    ];
    const curvePath = new THREE.CatmullRomCurve3(points);
    const tubeGeo = new THREE.TubeGeometry(curvePath, 100, 1.5, 8, false);
    const cableMesh = new THREE.Mesh(tubeGeo, cableMat);
    bridgeGroup.add(cableMesh);

    const lightInterval = 8;
    const pointEveryN = 3;
    for (let i = lightInterval; i < points.length; i += lightInterval) {
      const p = points[i]!;
      const addPt = (i % (lightInterval * pointEveryN) === 0);
      makeBulb(bridgeGroup, new THREE.Vector3(p.x, p.y - 1.5, p.z), addPt);
    }

    return points;
  };

  const leftCablePoints = createMainCable(15);
  const rightCablePoints = createMainCable(-15);

  const suspenderCount = leftCablePoints.length + rightCablePoints.length;
  const suspenderGeo = new THREE.CylinderGeometry(0.3, 0.3, 1, 8);
  const suspenderMesh = new THREE.InstancedMesh(suspenderGeo, cableMat, suspenderCount);
  const dummy = new THREE.Object3D();
  let idx = 0;
  [leftCablePoints, rightCablePoints].forEach(points => {
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

// --- Scene init ---
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
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  canvasContainer.value.appendChild(renderer.domElement);

  controls = new OrbitControls(camera, renderer.domElement);
  controls.maxPolarAngle = Math.PI * 0.495;
  controls.target.set(0, 10, 0);
  controls.minDistance = 40.0;
  controls.maxDistance = 2000.0;
  controls.update();

  // Sky
  sun = new THREE.Vector3();
  skyMesh = new Sky();
  skyMesh.scale.setScalar(10000);
  scene.add(skyMesh);
  skyUniforms = (skyMesh.material as THREE.ShaderMaterial).uniforms;
  skyUniforms['turbidity']!.value = 10;
  skyUniforms['rayleigh']!.value = 2;
  skyUniforms['mieCoefficient']!.value = 0.005;
  skyUniforms['mieDirectionalG']!.value = 0.8;

  pmremGenerator = new THREE.PMREMGenerator(renderer);
  sceneEnv = new THREE.Scene();

  // Water
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
      fog: scene.fog !== undefined
    }
  );
  water.rotation.x = -Math.PI / 2;
  scene.add(water);
  waterUniforms = (water.material as THREE.ShaderMaterial).uniforms;

  // Lighting
  ambientLight = new THREE.AmbientLight(0xcccccc, 0.4);
  scene.add(ambientLight);

  sunDirectional = new THREE.DirectionalLight(0xffaa33, 1);
  sunDirectional.position.set(-1, 1, 1);
  scene.add(sunDirectional);

  fogExp = new THREE.FogExp2(0xefd1b5, 0.0015);
  scene.fog = fogExp;

  buildBridge();

  window.addEventListener('resize', onWindowResize);

  lastClockTime = performance.now() / 1000;
  lastPMREMTime = lastClockTime - PMREM_MIN_INTERVAL - 1; // allow immediate first rebuild
  updateSceneForTime(timeOfDay.value, lastClockTime, true);

  animate();
};

// --- Animation loop ---
const animate = (): void => {
  animationId = requestAnimationFrame(animate);
  const now = performance.now() / 1000;
  const dt = now - lastClockTime;
  lastClockTime = now;

  if (isPlaying.value && !isDragging.value) {
    const hoursPerSecond = currentSpeed.value;
    let newMinutes = timeMinutes.value + dt * hoursPerSecond * 60;
    if (newMinutes >= MINUTES_PER_DAY) newMinutes = newMinutes % MINUTES_PER_DAY;
    if (newMinutes < 0) newMinutes += MINUTES_PER_DAY;
    timeMinutes.value = newMinutes;
  }

  if (water) {
    waterUniforms['time']!.value += dt;
  }

  const force = timeNeedsForceUpdate.value;
  if (force) timeNeedsForceUpdate.value = false;
  updateSceneForTime(timeOfDay.value, now, force);

  controls.update();
  renderer.render(scene, camera);
};

// --- UI Event handlers ---
function onSliderInput(e: Event): void {
  const target = e.target as HTMLInputElement;
  timeMinutes.value = parseInt(target.value, 10);
}
function onSliderStart(): void {
  isDragging.value = true;
}
function onSliderEnd(): void {
  isDragging.value = false;
  timeNeedsForceUpdate.value = true;
}
function togglePlay(): void {
  isPlaying.value = !isPlaying.value;
}
function cycleSpeed(): void {
  speedIndex.value = (speedIndex.value + 1) % SPEED_LEVELS.length;
}
function jumpTo(hours: number): void {
  timeMinutes.value = hours * 60;
  timeNeedsForceUpdate.value = true;
}

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

  <div class="time-panel" @mousedown.stop @touchstart.stop>
    <div class="time-row">
      <span class="time-label">{{ timeDisplay }}</span>
      <input
        type="range"
        class="time-slider"
        min="0"
        :max="MINUTES_PER_DAY"
        step="1"
        :value="timeMinutes"
        @input="onSliderInput"
        @mousedown="onSliderStart"
        @mouseup="onSliderEnd"
        @touchstart="onSliderStart"
        @touchend="onSliderEnd"
      />
    </div>
    <div class="btn-row">
      <button class="ctrl-btn play-btn" @click="togglePlay">
        <span v-if="isPlaying">❚❚</span>
        <span v-else>▶</span>
      </button>
      <button class="ctrl-btn speed-btn" @click="cycleSpeed">
        {{ currentSpeed }}×
      </button>
      <button class="ctrl-btn preset-btn" @click="jumpTo(PRESET_SUNRISE)">日出</button>
      <button class="ctrl-btn preset-btn" @click="jumpTo(PRESET_NOON)">正午</button>
      <button class="ctrl-btn preset-btn" @click="jumpTo(PRESET_SUNSET)">日落</button>
      <button class="ctrl-btn preset-btn" @click="jumpTo(PRESET_MIDNIGHT)">午夜</button>
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

.time-panel {
  position: fixed;
  right: 20px;
  bottom: 20px;
  z-index: 10;
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 12px 14px;
  background: rgba(0, 0, 0, 0.55);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  font-family: 'Helvetica Neue', Arial, sans-serif;
  color: #fff;
  min-width: 260px;
  max-width: 340px;
  user-select: none;
  -webkit-user-select: none;
  box-shadow: 0 6px 24px rgba(0,0,0,0.4);
}

.time-row {
  display: flex;
  align-items: center;
  gap: 10px;
}

.time-label {
  font-size: 1rem;
  font-weight: 500;
  font-variant-numeric: tabular-nums;
  min-width: 50px;
  letter-spacing: 1px;
  text-shadow: 0 1px 2px rgba(0,0,0,0.5);
}

.time-slider {
  flex: 1;
  -webkit-appearance: none;
  appearance: none;
  height: 4px;
  background: linear-gradient(to right, #0d1b3e, #1a2a5e, #f5a623, #ff6b35, #f5a623, #1a2a5e, #0d1b3e);
  border-radius: 2px;
  outline: none;
  cursor: pointer;
}

.time-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: #fff;
  border: 2px solid rgba(255, 200, 100, 0.9);
  box-shadow: 0 0 8px rgba(255, 200, 100, 0.6);
  cursor: pointer;
  transition: transform 0.1s;
}
.time-slider::-webkit-slider-thumb:active {
  transform: scale(1.2);
}
.time-slider::-moz-range-thumb {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: #fff;
  border: 2px solid rgba(255, 200, 100, 0.9);
  box-shadow: 0 0 8px rgba(255, 200, 100, 0.6);
  cursor: pointer;
}

.btn-row {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-wrap: wrap;
}

.ctrl-btn {
  background: rgba(255, 255, 255, 0.08);
  color: #fff;
  border: 1px solid rgba(255, 255, 255, 0.15);
  border-radius: 8px;
  padding: 6px 10px;
  font-size: 0.82rem;
  font-family: inherit;
  cursor: pointer;
  transition: background 0.15s, transform 0.1s;
  letter-spacing: 0.5px;
  min-width: 36px;
}

.ctrl-btn:hover {
  background: rgba(255, 255, 255, 0.18);
}
.ctrl-btn:active {
  transform: scale(0.95);
}

.play-btn {
  width: 36px;
  height: 32px;
  padding: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.75rem;
  background: rgba(255, 180, 60, 0.25);
  border-color: rgba(255, 180, 60, 0.5);
}
.play-btn:hover {
  background: rgba(255, 180, 60, 0.4);
}

.speed-btn {
  font-weight: 600;
  min-width: 48px;
}

.preset-btn {
  flex: 1;
  min-width: 42px;
  font-size: 0.78rem;
}

@media (max-width: 600px) {
  .time-panel {
    right: 10px;
    bottom: 10px;
    left: 10px;
    max-width: none;
    padding: 10px 12px;
  }
  .overlay {
    bottom: 90px;
    left: 16px;
  }
  .overlay h1 {
    font-size: 1.6rem;
  }
  .ctrl-btn {
    padding: 6px 6px;
    font-size: 0.75rem;
  }
}
</style>
