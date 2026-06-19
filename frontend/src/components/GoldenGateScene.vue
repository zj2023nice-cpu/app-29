<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { Water } from 'three/examples/jsm/objects/Water.js';
import { Sky } from 'three/examples/jsm/objects/Sky.js';

const canvasContainer = ref<HTMLDivElement | null>(null);

// Configuration
const BRIDGE_COLOR = 0xF04A00; // International Orange
const ROAD_COLOR = 0x333333;
const CABLE_COLOR = 0xF04A00;

// San Francisco location & time-control constants
const LAT_SF_DEG = 37.7749;
const LON_SF_DEG = -122.4194;
// San Francisco observes PDT (UTC-7) during daylight saving (mid-Mar..early-Nov);
// PST (UTC-8) the rest of the year. Today's date is in DST window so use -7.
const TIMEZONE_OFFSET_HOURS = -7;
const SECONDS_PER_HOUR = 3600;
const PMREM_UPDATE_DEG = 4; // Only refresh env map when sun altitude shifts by this many degrees
const LAMP_BASE_INTENSITY = 1.6;
const UI_SYNC_INTERVAL_MS = 200; // Throttle ref writes so Vue does not re-render every frame

// Pre-allocated reusable scratch objects (avoid per-frame allocation -> GC pressure)
const TMP_DIR = new THREE.Vector3();
const TMP_SUN_COLOR = new THREE.Color();
const TMP_FOG_COLOR = new THREE.Color();
const TMP_WATER_COLOR = new THREE.Color();
const TMP_EMISSIVE = new THREE.Color();
const FOG_DAY = new THREE.Color(0xefd1b5);
const FOG_NIGHT = new THREE.Color(0x05080f);
const FOG_DUSK = new THREE.Color(0xff7a3d);
const WATER_DAY = new THREE.Color(0x103a5a);
const WATER_NIGHT = new THREE.Color(0x000308);
const WATER_DUSK = new THREE.Color(0x4a1a0a);
const LAMP_COLOR = new THREE.Color(0xffb070);

let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let water: Water;
let sun: THREE.Vector3;
let animationId: number;

// Lighting / materials kept module-scoped so applyTimeOfDay can mutate them
let dirLight: THREE.DirectionalLight;
let ambientLight: THREE.AmbientLight;
let towerMatRef: THREE.MeshStandardMaterial;
let cableMatRef: THREE.MeshStandardMaterial;
const bridgeLights: THREE.PointLight[] = [];

// Sky / PMREM refs for time-of-day updates
let skyRef: Sky;
let pmremGeneratorRef: THREE.PMREMGenerator;
let sceneEnvRef: THREE.Scene;
let renderTargetRef: THREE.WebGLRenderTarget | null = null;
let lastPmremSunDeg = -999;
let lastFrameTime = 0;
let lastUiSyncTime = 0;

// Authoritative animation clock lives outside Vue's reactivity to avoid per-frame re-renders.
let currentTimeOfDay = 12;

// Reactive UI state (display-only; written at most every UI_SYNC_INTERVAL_MS)
const timeOfDay = ref<number>(12);
const isPlaying = ref<boolean>(true);
const speedFactor = ref<number>(100);

// Cleanup helper
const cleanUp = (): void => {
  if (animationId) cancelAnimationFrame(animationId);
  if (renderer) renderer.dispose();
  if (controls) controls.dispose();
  if (renderTargetRef) {
    renderTargetRef.dispose();
    renderTargetRef = null;
  }
  window.removeEventListener('resize', onWindowResize);
};

const onWindowResize = (): void => {
  if (!camera || !renderer) return;
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
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

// NOAA-style solar position for San Francisco. Writes the direction into TMP_DIR in place
// (Y up, -Z = north) and returns altitude in radians. Continuous over 0..24h via mod 1440.
function computeSunDirection(hour: number): { dir: THREE.Vector3; altitudeRad: number } {
  const now = new Date();
  const start = new Date(now.getFullYear(), 0, 0);
  const dayOfYear = Math.floor((now.getTime() - start.getTime()) / 86400000);

  const gamma = (2 * Math.PI / 365) * (dayOfYear - 1 + (hour - 12) / 24);
  const eqTime =
    229.18 *
    (0.000075 +
      0.001868 * Math.cos(gamma) -
      0.032077 * Math.sin(gamma) -
      0.014615 * Math.cos(2 * gamma) -
      0.040849 * Math.sin(2 * gamma)); // minutes
  const decl =
    0.006918 -
    0.399912 * Math.cos(gamma) +
    0.070257 * Math.sin(gamma) -
    0.006758 * Math.cos(2 * gamma) +
    0.000907 * Math.sin(2 * gamma) -
    0.002697 * Math.cos(3 * gamma) +
    0.00148 * Math.sin(3 * gamma); // radians

  const timeOffset = eqTime + 4 * LON_SF_DEG - 60 * TIMEZONE_OFFSET_HOURS; // minutes
  const tst = ((hour * 60 + timeOffset) % 1440 + 1440) % 1440; // true solar time, minutes
  const haDeg = tst / 4 - 180;
  const ha = (haDeg * Math.PI) / 180;

  const latRad = (LAT_SF_DEG * Math.PI) / 180;
  const sinAlt =
    Math.sin(latRad) * Math.sin(decl) +
    Math.cos(latRad) * Math.cos(decl) * Math.cos(ha);
  const altitude = Math.asin(THREE.MathUtils.clamp(sinAlt, -1, 1));

  const cosAz =
    (Math.sin(decl) - Math.sin(altitude) * Math.sin(latRad)) /
    (Math.cos(altitude) * Math.cos(latRad));
  let azimuth = Math.acos(THREE.MathUtils.clamp(cosAz, -1, 1));
  if (ha > 0) azimuth = 2 * Math.PI - azimuth; // morning -> east, afternoon -> west

  TMP_DIR.set(
    Math.sin(azimuth) * Math.cos(altitude),
    Math.sin(altitude),
    -Math.cos(azimuth) * Math.cos(altitude)
  ).normalize();

  return { dir: TMP_DIR, altitudeRad: altitude };
}

function applyTimeOfDay(hour: number): void {
  const { dir, altitudeRad } = computeSunDirection(hour);
  sun.copy(dir);

  const sinAlt = Math.sin(altitudeRad);
  const dayFactor = THREE.MathUtils.smoothstep(sinAlt, -0.1, 0.25); // 0=night, 1=day
  const duskFactor = 1 - Math.min(1, Math.abs(sinAlt) / 0.2); // peaks at horizon

  // Sky shader
  const skyU = (skyRef.material as THREE.ShaderMaterial).uniforms;
  skyU['sunPosition']!.value.copy(sun);
  skyU['turbidity']!.value = THREE.MathUtils.lerp(2, 12, duskFactor);
  skyU['rayleigh']!.value = THREE.MathUtils.lerp(0.5, 3, 1 - dayFactor * 0.5);
  skyU['mieCoefficient']!.value = THREE.MathUtils.lerp(0.002, 0.02, duskFactor);
  skyU['mieDirectionalG']!.value = 0.8;

  // Tone mapping exposure: dim at night, bright at day
  renderer.toneMappingExposure = THREE.MathUtils.lerp(0.05, 0.5, dayFactor);

  // Fog color & density (reuse TMP_FOG_COLOR; no allocation)
  TMP_FOG_COLOR.copy(FOG_NIGHT).lerp(FOG_DAY, dayFactor).lerp(FOG_DUSK, duskFactor * 0.6);
  const fog = scene.fog as THREE.FogExp2;
  fog.color.copy(TMP_FOG_COLOR);
  fog.density = THREE.MathUtils.lerp(0.0028, 0.0012, dayFactor);

  // Water uniforms (reuse TMP_WATER_COLOR / TMP_SUN_COLOR)
  const waterU = (water.material as THREE.ShaderMaterial).uniforms;
  waterU['sunDirection']!.value.copy(sun);
  TMP_WATER_COLOR.copy(WATER_NIGHT).lerp(WATER_DAY, dayFactor).lerp(WATER_DUSK, duskFactor * 0.5);
  waterU['waterColor']!.value.copy(TMP_WATER_COLOR);
  TMP_SUN_COLOR.setHSL(
    THREE.MathUtils.lerp(0.05, 0.13, dayFactor),
    1,
    THREE.MathUtils.lerp(0.4, 0.85, dayFactor)
  );
  waterU['sunColor']!.value.copy(TMP_SUN_COLOR);
  waterU['distortionScale']!.value = 3.7;

  // Directional + ambient light
  dirLight.position.copy(sun).multiplyScalar(500);
  dirLight.color.copy(TMP_SUN_COLOR);
  dirLight.intensity = Math.max(sinAlt, 0) * 1.2;
  ambientLight.color.setRGB(0.4 + 0.4 * dayFactor, 0.45 + 0.35 * dayFactor, 0.55);
  ambientLight.intensity = THREE.MathUtils.lerp(0.18, 0.45, dayFactor);

  // Bridge lamps: lit at night, off at day (reuse TMP_EMISSIVE)
  const lampLevel = 1 - THREE.MathUtils.smoothstep(sinAlt, -0.05, 0.15);
  for (let i = 0; i < bridgeLights.length; i++) {
    const l = bridgeLights[i]!;
    l.intensity = lampLevel * (l.userData.baseIntensity as number);
  }
  TMP_EMISSIVE.copy(LAMP_COLOR).multiplyScalar(lampLevel);
  towerMatRef.emissive.copy(TMP_EMISSIVE);
  towerMatRef.emissiveIntensity = lampLevel * 0.6;
  cableMatRef.emissive.copy(TMP_EMISSIVE);
  cableMatRef.emissiveIntensity = lampLevel * 0.4;

  // PMREM env map throttled by altitude delta
  const sunDeg = (altitudeRad * 180) / Math.PI;
  if (Math.abs(sunDeg - lastPmremSunDeg) > PMREM_UPDATE_DEG) {
    lastPmremSunDeg = sunDeg;
    if (renderTargetRef) renderTargetRef.dispose();
    sceneEnvRef.add(skyRef);
    renderTargetRef = pmremGeneratorRef.fromScene(sceneEnvRef);
    scene.add(skyRef);
    scene.environment = renderTargetRef.texture;
  }
}

function formatHHMM(h: number): string {
  const wrapped = ((h % 24) + 24) % 24;
  const hh = Math.floor(wrapped);
  const mm = Math.floor((wrapped - hh) * 60);
  return `${String(hh).padStart(2, '0')}:${String(mm).padStart(2, '0')}`;
}

function setTime(h: number): void {
  const wrapped = ((h % 24) + 24) % 24;
  currentTimeOfDay = wrapped;
  timeOfDay.value = wrapped;
  lastUiSyncTime = performance.now();
}

function togglePlay(): void {
  isPlaying.value = !isPlaying.value;
}

function setSpeed(s: number): void {
  speedFactor.value = s;
}

function onSliderInput(e: Event): void {
  const v = parseFloat((e.target as HTMLInputElement).value);
  if (!Number.isNaN(v)) setTime(v);
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
  const sky = new Sky();
  sky.scale.setScalar(10000);
  scene.add(sky);
  skyRef = sky;

  const skyUniforms = (sky.material as THREE.ShaderMaterial).uniforms;
  skyUniforms['turbidity']!.value = 10;
  skyUniforms['rayleigh']!.value = 2;
  skyUniforms['mieCoefficient']!.value = 0.005;
  skyUniforms['mieDirectionalG']!.value = 0.8;

  pmremGeneratorRef = new THREE.PMREMGenerator(renderer);
  sceneEnvRef = new THREE.Scene();

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
      waterColor: 0x001e0f,
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
  scene.fog = new THREE.FogExp2(0xefd1b5, 0.0015); // Matches the sunset-ish vibe

  // 8. Build The Bridge
  buildBridge();

  // 9. Apply initial time-of-day
  currentTimeOfDay = timeOfDay.value;
  applyTimeOfDay(currentTimeOfDay);

  // Event Listeners
  window.addEventListener('resize', onWindowResize);

  // Start Loop
  animate();
};

const buildBridge = (): void => {
  const bridgeGroup = new THREE.Group();
  scene.add(bridgeGroup);

  towerMatRef = new THREE.MeshStandardMaterial({
    color: BRIDGE_COLOR,
    roughness: 0.7,
    metalness: 0.1,
    emissive: 0x000000,
    emissiveIntensity: 0
  });

  const roadMat = new THREE.MeshStandardMaterial({
    color: ROAD_COLOR,
    roughness: 0.9
  });

  cableMatRef = new THREE.MeshStandardMaterial({
    color: CABLE_COLOR,
    roughness: 0.5,
    metalness: 0.2,
    emissive: 0x000000,
    emissiveIntensity: 0
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
    const legLeft = new THREE.Mesh(legGeo, towerMatRef);
    legLeft.position.set(0, towerHeight / 2, 15);
    legLeft.castShadow = true;
    legLeft.receiveShadow = true;

    const legRight = new THREE.Mesh(legGeo, towerMatRef);
    legRight.position.set(0, towerHeight / 2, -15);
    legRight.castShadow = true;
    legRight.receiveShadow = true;

    // Cross braces (Art Deco style)
    const braceGeo = new THREE.BoxGeometry(towerWidth - 2, 4, 30);
    const brace1 = new THREE.Mesh(braceGeo, towerMatRef);
    brace1.position.set(0, towerHeight * 0.9, 0);

    const brace2 = new THREE.Mesh(braceGeo, towerMatRef);
    brace2.position.set(0, towerHeight * 0.7, 0);

    const brace3 = new THREE.Mesh(braceGeo, towerMatRef);
    brace3.position.set(0, towerHeight * 0.5, 0);

    const brace4 = new THREE.Mesh(braceGeo, towerMatRef);
    brace4.position.set(0, deckY + 5, 0); // Below deck

    // Decorative top
    const topGeo = new THREE.BoxGeometry(towerWidth - 2, 10, towerDepth - 2);
    const topLeft = new THREE.Mesh(topGeo, towerMatRef);
    topLeft.position.set(0, towerHeight + 5, 15);
    const topRight = new THREE.Mesh(topGeo, towerMatRef);
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
    const cableMesh = new THREE.Mesh(tubeGeo, cableMatRef);
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
  const suspenderMesh = new THREE.InstancedMesh(suspenderGeo, cableMatRef, suspenderCount);

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

  // --- Bridge lamps (low-cost: a handful of PointLights) ---
  const lampPositions: Array<[number, number, number]> = [
    [-span / 2, towerHeight + 6, 0],
    [span / 2, towerHeight + 6, 0],
    [0, deckY + 8, 0],
    [-span / 2 - sideSpan, deckY + 4, 0],
    [span / 2 + sideSpan, deckY + 4, 0]
  ];
  for (let i = 0; i < lampPositions.length; i++) {
    const pos = lampPositions[i]!;
    const p = new THREE.PointLight(0xffb070, 0, 240, 2);
    p.position.set(pos[0], pos[1], pos[2]);
    p.userData.baseIntensity = LAMP_BASE_INTENSITY;
    bridgeGroup.add(p);
    bridgeLights.push(p);
  }
};

const animate = (): void => {
  animationId = requestAnimationFrame(animate);

  const now = performance.now();
  const delta = lastFrameTime ? (now - lastFrameTime) / 1000 : 0;
  lastFrameTime = now;

  if (isPlaying.value) {
    const next = currentTimeOfDay + (delta * speedFactor.value) / SECONDS_PER_HOUR;
    currentTimeOfDay = ((next % 24) + 24) % 24;
    // Throttle reactive write so Vue only re-renders the HUD a few times per second.
    if (now - lastUiSyncTime > UI_SYNC_INTERVAL_MS) {
      lastUiSyncTime = now;
      timeOfDay.value = currentTimeOfDay;
    }
  }

  applyTimeOfDay(currentTimeOfDay);

  if (water) {
    (water.material as THREE.ShaderMaterial).uniforms['time']!.value += 1.0 / 60.0;
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
  <div
    class="time-controls"
    @pointerdown.stop
    @touchstart.stop
    @wheel.stop
  >
    <div class="tc-row tc-top">
      <button class="tc-btn tc-play" @click="togglePlay">
        {{ isPlaying ? '⏸' : '▶' }}
      </button>
      <input
        class="tc-slider"
        type="range"
        min="0"
        max="24"
        step="0.01"
        :value="timeOfDay"
        @input="onSliderInput"
      />
      <span class="tc-time">{{ formatHHMM(timeOfDay) }}</span>
    </div>
    <div class="tc-row tc-bottom">
      <button :class="['tc-btn', speedFactor === 1 ? 'on' : '']" @click="setSpeed(1)">1×</button>
      <button :class="['tc-btn', speedFactor === 10 ? 'on' : '']" @click="setSpeed(10)">10×</button>
      <button :class="['tc-btn', speedFactor === 100 ? 'on' : '']" @click="setSpeed(100)">100×</button>
      <span class="tc-sep"></span>
      <button class="tc-btn" @click="setTime(6)">日出</button>
      <button class="tc-btn" @click="setTime(12)">正午</button>
      <button class="tc-btn" @click="setTime(18.5)">日落</button>
      <button class="tc-btn" @click="setTime(0)">午夜</button>
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

.time-controls {
  position: absolute;
  right: 16px;
  bottom: 16px;
  z-index: 5;
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 10px 12px;
  border-radius: 12px;
  background: rgba(15, 18, 28, 0.55);
  backdrop-filter: blur(6px);
  -webkit-backdrop-filter: blur(6px);
  color: #fff;
  font: 13px/1.2 -apple-system, 'Segoe UI', Roboto, sans-serif;
  user-select: none;
  touch-action: manipulation;
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.35);
  max-width: min(92vw, 420px);
}

.tc-row {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.tc-slider {
  flex: 1 1 160px;
  accent-color: #F04A00;
  height: 24px;
}

.tc-time {
  min-width: 46px;
  text-align: right;
  font-variant-numeric: tabular-nums;
  opacity: 0.9;
}

.tc-btn {
  appearance: none;
  border: 1px solid rgba(255, 255, 255, 0.18);
  background: rgba(255, 255, 255, 0.06);
  color: #fff;
  padding: 6px 10px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 13px;
  min-width: 40px;
  min-height: 32px;
}

.tc-btn.on {
  background: #F04A00;
  border-color: #F04A00;
}

.tc-btn.tc-play {
  font-size: 14px;
  min-width: 36px;
}

.tc-sep {
  width: 1px;
  height: 18px;
  background: rgba(255, 255, 255, 0.2);
  margin: 0 2px;
}

@media (max-width: 480px) {
  .tc-slider {
    flex: 1 1 100%;
  }
}
</style>
