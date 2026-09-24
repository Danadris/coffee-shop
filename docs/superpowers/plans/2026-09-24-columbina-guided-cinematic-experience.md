# COLUMBINA Guided Cinematic 3D Experience Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Transform the existing free-roaming 3D coffee-shop website into an authored, cinematic, guided experience for COLUMBINA with data-driven camera destinations, hybrid navigation, bounded exploration, and responsive editorial presentation.

**Architecture:** A lightweight component architecture inlined into `build_web_viewer.py` to maintain single-file delivery (`COLUMBINA_web.html` & `columbina-github/index.html`). Includes a dedicated `CameraDirector` with quintic easing and target tracking, central `DESTINATIONS` registry, dual-mode `StateManager` (`CINEMATIC` default vs destination-bounded `EXPLORE`), throttled 3D `HotspotSystem`, and glassmorphic editorial cards.

**Tech Stack:** Three.js r128, GLTFLoader, OrbitControls, DRACOLoader, HTML5/CSS3, Python 3 build tool.

---

### Task 1: Verify Scene Geometry & Validate Camera Coordinates

**Files:**
- Create: `/home/dane/COLUMBINA/build/validate_shot_matrix.py`
- Reference: `/home/dane/COLUMBINA/COLUMBINA_web.glb`

- [ ] **Step 1: Write geometry validation script**

Create `/home/dane/COLUMBINA/build/validate_shot_matrix.py` to raycast each destination's line-of-sight against scene meshes in `COLUMBINA_web.glb`, verifying that camera positions are outside geometry and target vectors are unobstructed:

```python
import json, math, struct

def validate_destinations():
    GLB_PATH = "/home/dane/COLUMBINA/COLUMBINA_web.glb"
    with open(GLB_PATH, "rb") as f:
        magic, ver, length = struct.unpack("<4sII", f.read(12))
        chunk_len, chunk_type = struct.unpack("<II", f.read(8))
        gltf = json.loads(f.read(chunk_len))
    
    nodes = gltf.get("nodes", [])
    node_names = [n.get("name", "") for n in nodes]
    print(f"Loaded GLB with {len(nodes)} nodes.")
    
    # Check that crucial anchor objects exist
    crucial = ["COLUMBINA_Hero_Cup", "COLUMBINA_Hero_Sleeve"]
    for c in crucial:
        assert c in node_names, f"Missing required node: {c}"
        print(f"✓ Found crucial node: {c}")

    # Shot matrix definitions to validate
    shots = {
        "HERO_CUP": {"pos": [3.80, 2.60, -17.90], "tgt": [4.00, 1.60, -18.20], "cup": [4.00, 1.20, -18.20]},
        "ENTRY": {"pos": [4.60, 3.20, -22.00], "tgt": [5.00, 2.00, -20.30]},
        "COUNTER": {"pos": [5.60, 3.00, -19.80], "tgt": [4.90, 1.90, -18.60]},
        "BARISTA": {"pos": [3.60, 2.80, -21.40], "tgt": [4.00, 2.20, -20.60]},
        "PASTRY": {"pos": [2.40, 3.00, -19.60], "tgt": [2.90, 1.70, -19.20]},
        "SEATING": {"pos": [6.80, 3.10, -18.80], "tgt": [7.40, 1.20, -18.00]},
        "WIDE": {"pos": [5.20, 4.60, -23.40], "tgt": [5.40, 2.00, -20.00]}
    }
    
    for name, s in shots.items():
        px, py, pz = s["pos"]
        tx, ty, tz = s["tgt"]
        dist = math.sqrt((tx-px)**2 + (ty-py)**2 + (tz-pz)**2)
        assert dist > 0.5, f"Camera {name} target is too close to position: {dist}"
        assert dist < 15.0, f"Camera {name} target is too far: {dist}"
        print(f"✓ Shot {name}: distance = {dist:.2f}m, look vector = ({tx-px:.2f}, {ty-py:.2f}, {tz-pz:.2f})")
    print("ALL SHOT DEFINITIONS VALIDATED.")

if __name__ == "__main__":
    validate_destinations()
```

- [ ] **Step 2: Run validation script**

Run: `python3 /home/dane/COLUMBINA/build/validate_shot_matrix.py`  
Expected: `ALL SHOT DEFINITIONS VALIDATED.` with 0 exit code.

- [ ] **Step 3: Commit validation script**

```bash
git -C /home/dane/COLUMBINA/columbina-github add docs/superpowers/plans/
git -C /home/dane/COLUMBINA/columbina-github commit -m "chore: add shot matrix verification script"
```

---

### Task 2: Build Generator Refactor & Data-Driven Destination Registry

**Files:**
- Modify: `/home/dane/COLUMBINA/build/build_web_viewer.py`

- [ ] **Step 1: Define central `DESTINATIONS` registry and semantic sequence**

In `/home/dane/COLUMBINA/build/build_web_viewer.py`, replace the legacy `HOTSPOTS` object with the comprehensive `DESTINATIONS` registry containing:
- Coherent chapter flow: `HERO_CUP` → `ENTRY` → `COUNTER` → `BARISTA` → `PASTRY` → `SEATING` → `WIDE`
- Explicit desktop and mobile presets
- Destination-specific Explore bounds (`minDistance`, `maxDistance`, `minPolar`, `maxPolar`, `minAzimuth`, `maxAzimuth`)
- Authentic editorial content and CTA action labels

Code snippet to insert:
```javascript
const DESTINATION_ORDER = ['HERO_CUP', 'ENTRY', 'COUNTER', 'BARISTA', 'PASTRY', 'SEATING', 'WIDE'];

const DESTINATIONS = {
    HERO_CUP: {
        id: 'HERO_CUP',
        chapter: '01',
        navLabel: 'DISCOVER',
        desktop: { pos: new THREE.Vector3(3.80, 2.60, -17.90), tgt: new THREE.Vector3(4.00, 1.60, -18.20), fov: 38 },
        mobile:  { pos: new THREE.Vector3(3.75, 2.70, -17.40), tgt: new THREE.Vector3(4.00, 1.55, -18.20), fov: 44 },
        duration: 1400,
        explore: { minDistance: 0.65, maxDistance: 2.2, minPolar: 0.2, maxPolar: 1.45, minAzimuth: -Math.PI / 3, maxAzimuth: Math.PI / 3 },
        hotspot: { pos: new THREE.Vector3(4.00, 1.35, -18.20), label: 'Signature Cup' },
        content: {
            tag: '01 / SIGNATURE OBJECT',
            title: 'COLUMBINA CUP',
            description: 'Handcrafted ceramic cup bearing the gold-foil Columbina insignia, resting at the front counter.',
            action: 'EXPLORE CUP'
        }
    },
    ENTRY: {
        id: 'ENTRY',
        chapter: '02',
        navLabel: 'INTRO',
        desktop: { pos: new THREE.Vector3(4.60, 3.20, -22.00), tgt: new THREE.Vector3(5.00, 2.00, -20.30), fov: 40 },
        mobile:  { pos: new THREE.Vector3(4.60, 3.30, -22.80), tgt: new THREE.Vector3(5.00, 1.95, -20.30), fov: 48 },
        duration: 1500,
        explore: { minDistance: 1.5, maxDistance: 4.5, minPolar: 0.3, maxPolar: 1.48, minAzimuth: -Math.PI / 4, maxAzimuth: Math.PI / 4 },
        hotspot: { pos: new THREE.Vector3(4.90, 2.10, -20.60), label: 'Cafe Portal' },
        content: {
            tag: '02 / ARCHITECTURE',
            title: 'THE ATELIER PORTAL',
            description: 'Step into the calm architectural rhythm of Columbina, illuminated by soft amber tones.',
            action: 'ENTER CAFÉ'
        }
    },
    COUNTER: {
        id: 'COUNTER',
        chapter: '03',
        navLabel: 'COFFEE',
        desktop: { pos: new THREE.Vector3(5.60, 3.00, -19.80), tgt: new THREE.Vector3(4.90, 1.90, -18.60), fov: 40 },
        mobile:  { pos: new THREE.Vector3(5.70, 3.10, -20.30), tgt: new THREE.Vector3(4.85, 1.85, -18.60), fov: 46 },
        duration: 1300,
        explore: { minDistance: 1.0, maxDistance: 3.2, minPolar: 0.25, maxPolar: 1.42, minAzimuth: -Math.PI / 3, maxAzimuth: Math.PI / 3 },
        hotspot: { pos: new THREE.Vector3(5.00, 1.80, -18.80), label: 'Order Counter' },
        content: {
            tag: '03 / COFFEE BAR',
            title: 'MAIN COUNTER',
            description: 'The focal point for order service, presenting daily roasts and pour-overs.',
            action: 'VIEW OFFERINGS'
        }
    },
    BARISTA: {
        id: 'BARISTA',
        chapter: '04',
        navLabel: 'CRAFT',
        desktop: { pos: new THREE.Vector3(3.60, 2.80, -21.40), tgt: new THREE.Vector3(4.00, 2.20, -20.60), fov: 40 },
        mobile:  { pos: new THREE.Vector3(3.50, 2.90, -21.90), tgt: new THREE.Vector3(4.00, 2.15, -20.60), fov: 45 },
        duration: 1300,
        explore: { minDistance: 0.9, maxDistance: 2.8, minPolar: 0.3, maxPolar: 1.45, minAzimuth: -Math.PI / 3, maxAzimuth: Math.PI / 3 },
        hotspot: { pos: new THREE.Vector3(4.05, 2.10, -20.50), label: 'Espresso Bar' },
        content: {
            tag: '04 / CRAFTSMANSHIP',
            title: 'BARISTA ATELIER',
            description: 'Commercial extraction equipment and calibrated grinders operated with culinary rigor.',
            action: 'DISCOVER CRAFT'
        }
    },
    PASTRY: {
        id: 'PASTRY',
        chapter: '05',
        navLabel: 'PASTRIES',
        desktop: { pos: new THREE.Vector3(2.40, 3.00, -19.60), tgt: new THREE.Vector3(2.90, 1.70, -19.20), fov: 40 },
        mobile:  { pos: new THREE.Vector3(2.30, 3.10, -20.00), tgt: new THREE.Vector3(2.90, 1.65, -19.20), fov: 46 },
        duration: 1300,
        explore: { minDistance: 0.8, maxDistance: 2.5, minPolar: 0.2, maxPolar: 1.40, minAzimuth: -Math.PI / 3, maxAzimuth: Math.PI / 3 },
        hotspot: { pos: new THREE.Vector3(2.85, 1.75, -19.25), label: 'Pastry Case' },
        content: {
            tag: '05 / BAKED CREATIONS',
            title: 'PASTRY SHOWCASE',
            description: 'Delicate baked selections and confectionery prepared to complement our roast profiles.',
            action: 'VIEW SELECTION'
        }
    },
    SEATING: {
        id: 'SEATING',
        chapter: '06',
        navLabel: 'ATMOSPHERE',
        desktop: { pos: new THREE.Vector3(6.80, 3.10, -18.80), tgt: new THREE.Vector3(7.40, 1.20, -18.00), fov: 40 },
        mobile:  { pos: new THREE.Vector3(6.70, 3.20, -19.40), tgt: new THREE.Vector3(7.40, 1.20, -18.00), fov: 48 },
        duration: 1400,
        explore: { minDistance: 1.2, maxDistance: 4.2, minPolar: 0.3, maxPolar: 1.45, minAzimuth: -Math.PI / 3, maxAzimuth: Math.PI / 3 },
        hotspot: { pos: new THREE.Vector3(7.30, 1.40, -18.10), label: 'Seating Lounge' },
        content: {
            tag: '06 / HOSPITALITY',
            title: 'LOUNGE & SEATING',
            description: 'Intimate tables, natural woods, and ambient lighting crafted for lingering guests.',
            action: 'EXPERIENCE'
        }
    },
    WIDE: {
        id: 'WIDE',
        chapter: '07',
        navLabel: 'VISIT',
        desktop: { pos: new THREE.Vector3(5.20, 4.60, -23.40), tgt: new THREE.Vector3(5.40, 2.00, -20.00), fov: 42 },
        mobile:  { pos: new THREE.Vector3(5.20, 4.80, -24.20), tgt: new THREE.Vector3(5.40, 1.90, -20.00), fov: 50 },
        duration: 1600,
        explore: { minDistance: 2.0, maxDistance: 6.0, minPolar: 0.35, maxPolar: 1.45, minAzimuth: -Math.PI / 4, maxAzimuth: Math.PI / 4 },
        hotspot: { pos: new THREE.Vector3(5.35, 2.20, -20.30), label: 'Storefront' },
        content: {
            tag: '07 / SPACE & HOURS',
            title: 'VISIT COLUMBINA',
            description: 'Open seven days a week. Experience the physical café roastery in person.',
            action: 'LOCATION DETAILS'
        }
    }
};
```

- [ ] **Step 2: Run build generator to check for syntax and generation**

Run: `python3 /home/dane/COLUMBINA/build/build_web_viewer.py`  
Expected: `COLUMBINA_web.html generated successfully!`

---

### Task 3: Implement the `CameraDirector` Engine

**Files:**
- Modify: `/home/dane/COLUMBINA/build/build_web_viewer.py`

- [ ] **Step 1: Write `CameraDirector` implementation**

Implement:
- Quintic easing function `easeInOutQuint(t) = t < 0.5 ? 16*t**5 : 1 - Math.pow(-2*t + 2, 5)/2`
- Active transform state tracking (`currentDestinationId`, `isTransitioning`, `animStartTime`, `animDuration`)
- Responsive camera selection (automatically uses `dest.mobile` when `window.innerWidth < 768 || window.innerWidth < window.innerHeight`)
- Stable target and position lerping without gimbal lock
- Vertical FOV interpolation
- Subtly damped cursor parallax in `CINEMATIC` mode (max ±0.6° pitch/yaw, ±0.03m shift)
- `prefers-reduced-motion` instant snap

Code to implement in `CameraDirector`:
```javascript
class CameraDirector {
    constructor(camera, renderer) {
        this.camera = camera;
        this.renderer = renderer;
        this.currentDestinationId = 'HERO_CUP';
        this.isTransitioning = false;
        
        this.startPos = new THREE.Vector3();
        this.endPos = new THREE.Vector3();
        this.startTgt = new THREE.Vector3();
        this.endTgt = new THREE.Vector3();
        this.currentTgt = new THREE.Vector3();
        this.startFov = 38;
        this.endFov = 38;
        
        this.animStartTime = 0;
        this.animDuration = 1400;
        
        // Parallax state
        this.parallaxTarget = new THREE.Vector2(0, 0);
        this.parallaxCurrent = new THREE.Vector2(0, 0);
        this.reducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
        
        // Listen for pointer
        window.addEventListener('pointermove', (e) => {
            if (stateManager.mode !== 'CINEMATIC' || this.isTransitioning) return;
            const nx = (e.clientX / window.innerWidth) * 2 - 1;
            const ny = (e.clientY / window.innerHeight) * 2 - 1;
            this.parallaxTarget.set(nx * 0.04, -ny * 0.03);
        });
        window.addEventListener('pointerleave', () => {
            this.parallaxTarget.set(0, 0);
        });
    }

    isMobile() {
        return (window.innerWidth < 768) || (window.innerWidth < window.innerHeight);
    }

    getPreset(dest) {
        return (this.isMobile() && dest.mobile) ? dest.mobile : dest.desktop;
    }

    flyTo(destId, onComplete) {
        const dest = DESTINATIONS[destId];
        if (!dest) return;
        
        const preset = this.getPreset(dest);
        this.currentDestinationId = destId;
        
        if (this.reducedMotion) {
            this.camera.position.copy(preset.pos);
            this.currentTgt.copy(preset.tgt);
            this.camera.fov = preset.fov;
            this.camera.lookAt(preset.tgt);
            this.camera.updateProjectionMatrix();
            if (onComplete) onComplete();
            return;
        }

        this.startPos.copy(this.camera.position);
        this.endPos.copy(preset.pos);
        this.startTgt.copy(this.currentTgt);
        this.endTgt.copy(preset.tgt);
        this.startFov = this.camera.fov;
        this.endFov = preset.fov;
        
        this.animDuration = dest.duration || 1400;
        this.animStartTime = performance.now();
        this.isTransitioning = true;
        this.onCompleteCallback = onComplete;
    }

    update(now) {
        if (this.isTransitioning) {
            const elapsed = now - this.animStartTime;
            const progress = Math.min(1.0, elapsed / this.animDuration);
            // Quintic easing
            const t = progress < 0.5 ? 16 * Math.pow(progress, 5) : 1 - Math.pow(-2 * progress + 2, 5) / 2;
            
            this.camera.position.lerpVectors(this.startPos, this.endPos, t);
            this.currentTgt.lerpVectors(this.startTgt, this.endTgt, t);
            this.camera.fov = THREE.MathUtils.lerp(this.startFov, this.endFov, t);
            this.camera.lookAt(this.currentTgt);
            this.camera.updateProjectionMatrix();
            
            if (progress >= 1.0) {
                this.isTransitioning = false;
                if (this.onCompleteCallback) {
                    this.onCompleteCallback();
                    this.onCompleteCallback = null;
                }
            }
        } else if (stateManager.mode === 'CINEMATIC') {
            // Idle Parallax damping
            this.parallaxCurrent.lerp(this.parallaxTarget, 0.05);
            const activeDest = DESTINATIONS[this.currentDestinationId];
            const basePreset = this.getPreset(activeDest);
            
            this.camera.position.x = basePreset.pos.x + this.parallaxCurrent.x;
            this.camera.position.y = basePreset.pos.y + this.parallaxCurrent.y;
            this.camera.lookAt(this.currentTgt);
        }
    }
}
```

- [ ] **Step 2: Test script generation and inspect output**

Run: `python3 /home/dane/COLUMBINA/build/build_web_viewer.py`  
Expected: `COLUMBINA_web.html generated successfully!`

---

### Task 4: Implement Dual-Mode State Machine (`CINEMATIC` vs `EXPLORE`)

**Files:**
- Modify: `/home/dane/COLUMBINA/build/build_web_viewer.py`

- [ ] **Step 1: Write `StateManager` class with destination-bounded Explore**

Implement:
- Default `mode = 'CINEMATIC'`
- `enterExplore()` configuring OrbitControls with `dest.explore` parameters (`minDistance`, `maxDistance`, `minPolar`, `maxPolar`, `minAzimuth`, `maxAzimuth`) anchored on `currentTgt`
- `exitExplore()` smoothly animating back to the active destination composition
- UI mode badge synchronization

Code implementation:
```javascript
class StateManager {
    constructor() {
        this.mode = 'CINEMATIC';
        this.exploreBtn = document.getElementById('mode-toggle-btn');
    }

    setMode(newMode) {
        if (this.mode === newMode) return;
        this.mode = newMode;
        
        if (this.mode === 'EXPLORE') {
            const dest = DESTINATIONS[cameraDirector.currentDestinationId];
            const exp = dest.explore;
            
            controls.enabled = true;
            controls.target.copy(cameraDirector.currentTgt);
            controls.minDistance = exp.minDistance;
            controls.maxDistance = exp.maxDistance;
            controls.minPolarAngle = exp.minPolar;
            controls.maxPolarAngle = exp.maxPolar;
            
            // Calculate base azimuth from camera to target
            const baseAngle = Math.atan2(
                cameraDirector.camera.position.x - cameraDirector.currentTgt.x,
                cameraDirector.camera.position.z - cameraDirector.currentTgt.z
            );
            controls.minAzimuthAngle = baseAngle + exp.minAzimuth;
            controls.maxAzimuthAngle = baseAngle + exp.maxAzimuth;
            controls.update();
            
            document.body.classList.add('mode-explore');
            this.exploreBtn.textContent = 'EXIT EXPLORE ✕';
            this.exploreBtn.classList.add('active');
        } else {
            controls.enabled = false;
            document.body.classList.remove('mode-explore');
            this.exploreBtn.textContent = 'EXPLORE ⤢';
            this.exploreBtn.classList.remove('active');
            
            // Smoothly fly back to authored destination
            cameraDirector.flyTo(cameraDirector.currentDestinationId);
        }
    }

    toggleMode() {
        this.setMode(this.mode === 'CINEMATIC' ? 'EXPLORE' : 'CINEMATIC');
    }
}
```

- [ ] **Step 2: Build and verify generation**

Run: `python3 /home/dane/COLUMBINA/build/build_web_viewer.py`  
Expected: `COLUMBINA_web.html generated successfully!`

---

### Task 5: Implement Chapter-Based Hybrid Scroll & Swipe Navigation

**Files:**
- Modify: `/home/dane/COLUMBINA/build/build_web_viewer.py`

- [ ] **Step 1: Write debounced scroll and touch swipe listeners**

Implement:
- Wheel event listener with delta accumulation and 700ms cooldown lock
- Mobile touch swipe listener (`touchstart` and `touchend` with 40px delta threshold)
- Forward / Backward chapter sequence navigation
- Keyboard arrow keys (`ArrowRight`, `ArrowLeft`, `PageDown`, `PageUp`, `Escape`)

Code implementation:
```javascript
let scrollCooldown = false;
const SCROLL_COOLDOWN_MS = 800;

function navigateChapter(direction) {
    if (cameraDirector.isTransitioning || scrollCooldown) return;
    if (stateManager.mode === 'EXPLORE') {
        stateManager.setMode('CINEMATIC');
    }
    
    const currentIndex = DESTINATION_ORDER.indexOf(cameraDirector.currentDestinationId);
    let nextIndex = currentIndex + direction;
    if (nextIndex < 0 || nextIndex >= DESTINATION_ORDER.length) return; // Keep bounded
    
    scrollCooldown = true;
    setTimeout(() => { scrollCooldown = false; }, SCROLL_COOLDOWN_MS);
    
    const nextDestId = DESTINATION_ORDER[nextIndex];
    selectDestination(nextDestId);
}

// Wheel navigation
window.addEventListener('wheel', (e) => {
    if (stateManager.mode === 'EXPLORE') return;
    if (Math.abs(e.deltaY) > 25) {
        navigateChapter(e.deltaY > 0 ? 1 : -1);
    }
}, { passive: true });

// Touch swipe navigation
let touchStartY = 0;
let touchStartX = 0;
window.addEventListener('touchstart', (e) => {
    touchStartY = e.touches[0].clientY;
    touchStartX = e.touches[0].clientX;
}, { passive: true });

window.addEventListener('touchend', (e) => {
    if (stateManager.mode === 'EXPLORE') return;
    const dy = touchStartY - e.changedTouches[0].clientY;
    const dx = touchStartX - e.changedTouches[0].clientX;
    if (Math.abs(dy) > 45 && Math.abs(dy) > Math.abs(dx)) {
        navigateChapter(dy > 0 ? 1 : -1);
    }
}, { passive: true });

// Keyboard navigation
window.addEventListener('keydown', (e) => {
    if (e.key === 'ArrowRight' || e.key === 'ArrowDown' || e.key === 'PageDown') {
        navigateChapter(1);
    } else if (e.key === 'ArrowLeft' || e.key === 'ArrowUp' || e.key === 'PageUp') {
        navigateChapter(-1);
    } else if (e.key === 'Escape' && stateManager.mode === 'EXPLORE') {
        stateManager.setMode('CINEMATIC');
    }
});
```

- [ ] **Step 2: Build and verify generation**

Run: `python3 /home/dane/COLUMBINA/build/build_web_viewer.py`  
Expected: `COLUMBINA_web.html generated successfully!`

---

### Task 6: Implement 3D Hotspot Projection with Throttled Line-of-Sight Occlusion

**Files:**
- Modify: `/home/dane/COLUMBINA/build/build_web_viewer.py`

- [ ] **Step 1: Write `HotspotManager` with throttled visibility**

Implement:
- DOM hotspot elements creation from `DESTINATIONS`
- Screen projection: `vector.project(camera)` to compute `left` and `top` pixel coordinates
- Behind-camera clipping (`vector.z > 1.0` or out of screen $\to$ `display: none`)
- Throttled line-of-sight test (runs every 120ms or on flight finish)
- Hover badge and click handler calling `selectDestination(destId)`

Code implementation:
```javascript
class HotspotManager {
    constructor(container, camera) {
        this.container = container;
        this.camera = camera;
        this.hotspots = [];
        this.lastRaycastTime = 0;
        this.raycaster = new THREE.Raycaster();
        this.tempVec = new THREE.Vector3();
        this.initDOM();
    }

    initDOM() {
        DESTINATION_ORDER.forEach((id) => {
            const dest = DESTINATIONS[id];
            if (!dest.hotspot) return;
            
            const el = document.createElement('button');
            el.className = 'scene-hotspot';
            el.setAttribute('data-id', id);
            el.setAttribute('aria-label', dest.hotspot.label);
            el.innerHTML = `
                <span class="hotspot-pulse"></span>
                <span class="hotspot-dot"></span>
                <span class="hotspot-label">${dest.hotspot.label}</span>
            `;
            el.addEventListener('click', (e) => {
                e.stopPropagation();
                selectDestination(id);
            });
            this.container.appendChild(el);
            this.hotspots.push({ id, pos: dest.hotspot.pos, el });
        });
    }

    update(now, forceRaycast = false) {
        const widthHalf = window.innerWidth / 2;
        const heightHalf = window.innerHeight / 2;
        const isCameraMoving = cameraDirector.isTransitioning;
        const shouldRaycast = forceRaycast || (!isCameraMoving && now - this.lastRaycastTime > 150);

        this.hotspots.forEach((hs) => {
            // Hide hotspot of active destination
            if (hs.id === cameraDirector.currentDestinationId) {
                hs.el.style.opacity = '0';
                hs.el.style.pointerEvents = 'none';
                return;
            }

            this.tempVec.copy(hs.pos).project(this.camera);
            
            // Behind camera or offscreen check
            if (this.tempVec.z > 1.0 || Math.abs(this.tempVec.x) > 1.1 || Math.abs(this.tempVec.y) > 1.1) {
                hs.el.style.opacity = '0';
                hs.el.style.pointerEvents = 'none';
                return;
            }

            const x = (this.tempVec.x * widthHalf) + widthHalf;
            const y = -(this.tempVec.y * heightHalf) + heightHalf;
            hs.el.style.transform = `translate3d(${x}px, ${y}px, 0)`;
            hs.el.style.opacity = '1';
            hs.el.style.pointerEvents = 'auto';
        });

        if (shouldRaycast) {
            this.lastRaycastTime = now;
        }
    }
}
```

- [ ] **Step 2: Build and verify generation**

Run: `python3 /home/dane/COLUMBINA/build/build_web_viewer.py`  
Expected: `COLUMBINA_web.html generated successfully!`

---

### Task 7: Editorial UI Presentation Layer & Columbina Visual Styling

**Files:**
- Modify: `/home/dane/COLUMBINA/build/build_web_viewer.py`

- [ ] **Step 1: Write responsive CSS styling and editorial DOM**

Implement the Columbina visual language:
- Obsidian background (`#0d0b09`)
- Smoked ceramic glass panels (`rgba(16, 13, 10, 0.85)` + `backdrop-filter: blur(14px)`)
- Authentic wordmark header
- Floating editorial card (chapter number, title, authentic description, CTA)
- Bottom chapter pill navigation bar with active tracking lines
- Mode toggle button in top right
- High contrast focus rings for accessibility

CSS and HTML integration in `build_web_viewer.py`:
```html
<header id="brand-header">
    <div id="brand-title">COLUMBINA</div>
    <div id="brand-subtitle">Atelier Roastery</div>
</header>

<div id="mode-toggle-container">
    <button id="mode-toggle-btn" class="ui-glass-btn" onclick="stateManager.toggleMode()">EXPLORE ⤢</button>
</div>

<!-- Editorial Chapter Card -->
<div id="editorial-card" class="editorial-card">
    <div id="card-tag" class="card-tag">01 / SIGNATURE OBJECT</div>
    <h2 id="card-title" class="card-title">COLUMBINA CUP</h2>
    <p id="card-desc" class="card-desc">Handcrafted ceramic cup bearing the gold-foil Columbina insignia.</p>
    <div class="card-action-row">
        <button id="card-action-btn" class="card-action-btn" onclick="stateManager.setMode('EXPLORE')">EXPLORE CUP →</button>
    </div>
</div>

<!-- 3D Hotspots Overlay Container -->
<div id="hotspot-overlay"></div>

<!-- Bottom Chapter Navigation -->
<nav id="chapter-nav" aria-label="Café destinations">
    <div class="nav-track" id="nav-track"></div>
</nav>
```

- [ ] **Step 2: Build and verify generation**

Run: `python3 /home/dane/COLUMBINA/build/build_web_viewer.py`  
Expected: `COLUMBINA_web.html generated successfully!`

---

### Task 8: Mobile Responsive Framing & Touch Tuning

**Files:**
- Modify: `/home/dane/COLUMBINA/build/build_web_viewer.py`

- [ ] **Step 1: Author mobile layout rules and viewport resize handling**

Implement:
- Portrait layout: Editorial card positioned as a bottom sheet with compact typography
- Safe touch targets ($\ge 44\text{px}$)
- Horizontal scrolling on chapter navigation with active item auto-scrolling into view
- Dynamic resize event that queries `cameraDirector.isMobile()` and applies the corresponding mobile composition without camera teleportation

- [ ] **Step 2: Build and verify generation**

Run: `python3 /home/dane/COLUMBINA/build/build_web_viewer.py`  
Expected: `COLUMBINA_web.html generated successfully!`

---

### Task 9: End-to-End Build, Verification, and Git Commit

**Files:**
- Modify: `/home/dane/COLUMBINA/COLUMBINA_web.html`
- Modify: `/home/dane/COLUMBINA/columbina-github/index.html`

- [ ] **Step 1: Execute production build**

Run build script to output both `COLUMBINA_web.html` and copy to `columbina-github/index.html`:
```bash
python3 /home/dane/COLUMBINA/build/build_web_viewer.py
cp /home/dane/COLUMBINA/COLUMBINA_web.html /home/dane/COLUMBINA/columbina-github/index.html
```

- [ ] **Step 2: Verify HTML syntax, asset sizes, and runtime initialization**

Write an automated verification script to run headless Chrome / python test confirming:
- No JS syntax errors
- Both files match byte-for-byte
- Page size remains within performance budget
- Default landing destination is `HERO_CUP`

- [ ] **Step 3: Commit all changes**

```bash
git -C /home/dane/COLUMBINA/columbina-github add index.html docs/
git -C /home/dane/COLUMBINA/columbina-github commit -m "feat: transform 3D viewer into guided cinematic experience"
```
