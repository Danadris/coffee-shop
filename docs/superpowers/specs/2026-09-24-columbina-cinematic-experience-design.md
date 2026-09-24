# COLUMBINA — Guided Cinematic 3D Experience Design Specification

**Date:** 2026-09-24  
**Project:** COLUMBINA Flagship 3D Website  
**Reference Philosophy:** *Caffè Design* (Guided 3D world as the website fabric)  
**Target Delivery:** Self-contained single-page application (`COLUMBINA_web.html` & `columbina-github/index.html`)  

---

## 1. Executive Summary & Brand Direction

COLUMBINA is a premium 3D digital flagship experience for an artisan coffee roastery and pastry atelier. This design transforms the existing free-roaming 3D café into a **curated, cinematic interactive website** where the visitor is guided through intentionally authored compositions.

### Core Visual Principles
- **Atmosphere:** Architectural, warm, sophisticated, restrained, and editorial.
- **Palette & Finishes:** 
  - Deep Obsidian Ground (`#0d0b09`) matching the café's interior materials.
  - Smoked Ceramic Glass (`rgba(18, 15, 12, 0.78)`, `backdrop-filter: blur(14px)`) with delicate hairline borders (`rgba(215, 192, 150, 0.18)`).
  - Warm Crema Text (`#f5eedf`) for clear typographic hierarchy.
- **Brand Integrity:** 
  - Uses authentic COLUMBINA branding and the physical in-scene cup sleeve logo.
  - No fabricated or unverified business claims.
  - Restrained typography and zero game-like HUD clutter.

---

## 2. System Architecture

The application is structured into decoupled, modular systems maintained inside `build_web_viewer.py` and output to `index.html` (single-file deliverable, zero runtime external dependencies):

```
┌────────────────────────────────────────────────────────────────────────┐
│                          COLUMBINA APP ENGINE                          │
├──────────────────┬───────────────────┬─────────────────────────────────┤
│  CameraDirector  │  DestinationData  │          StateManager           │
│  - Quintic lerp  │  - Desktop shots  │  - CINEMATIC (Default)          │
│  - Target track  │  - Mobile shots   │  - EXPLORE (Bounded)            │
│  - FOV transit   │  - Content copy   │  - Transition locks             │
│  - Idle parallax │  - Explore bounds │  - Input debouncing             │
├──────────────────┼───────────────────┼─────────────────────────────────┤
│  HotspotSystem   │    UI / Nav Bar   │       EditorialPresenter        │
│  - 3D projection │  - Chapter tabs   │  - Staggered glass cards        │
│  - Throttled ray │  - Mode toggle    │  - Story, details, action CTA   │
│  - Occlusion test│  - Keyboard a11y  │  - Responsive sheet on mobile   │
└──────────────────┴───────────────────┴─────────────────────────────────┘
```

### 2.1 Dependencies & Asset Pipeline
- **Engine:** Three.js r128 (inlined).
- **Loaders:** GLTFLoader + DRACOLoader with embedded self-contained Draco WASM wrapper.
- **Controls:** Modified OrbitControls restricted to `EXPLORE` mode.
- **Model:** `COLUMBINA_web.glb` (11.17 MB, 203 meshes, Draco compressed, containing the complete café interior, 33 ceramic cups, and the hero cup).

---

## 3. Data-Driven Destination Architecture

All physical viewpoints, mobile variants, content copy, and destination-specific exploration envelopes are registered centrally in `DESTINATIONS`:

```javascript
const DESTINATIONS = {
  HERO_CUP: {
    id: 'HERO_CUP',
    chapter: '01',
    navLabel: 'DISCOVER',
    desktop: {
      pos: new THREE.Vector3(3.80, 2.60, -17.90),
      tgt: new THREE.Vector3(4.00, 1.60, -18.20),
      fov: 38
    },
    mobile: {
      pos: new THREE.Vector3(3.70, 2.70, -17.40),
      tgt: new THREE.Vector3(4.00, 1.55, -18.20),
      fov: 44
    },
    duration: 1400,
    explore: {
      minDistance: 0.7,
      maxDistance: 2.2,
      minPolar: 0.2,
      maxPolar: 1.45,
      minAzimuth: -Math.PI / 3,
      maxAzimuth: Math.PI / 3
    },
    hotspot: { x: 4.00, y: 1.35, z: -18.20, label: 'Signature Cup' },
    content: {
      tag: '01 / THE SIGNATURE OBJECT',
      title: 'COLUMBINA CUP',
      description: 'Handcrafted ceramic branded cup, positioned at the heart of the counter.',
      actionLabel: 'EXPLORE CUP'
    }
  },
  ENTRY: {
    id: 'ENTRY',
    chapter: '02',
    navLabel: 'INTRO',
    desktop: {
      pos: new THREE.Vector3(4.60, 3.20, -22.00),
      tgt: new THREE.Vector3(5.00, 2.00, -20.30),
      fov: 40
    },
    mobile: {
      pos: new THREE.Vector3(4.60, 3.30, -22.80),
      tgt: new THREE.Vector3(5.00, 1.95, -20.30),
      fov: 48
    },
    duration: 1500,
    explore: {
      minDistance: 1.5,
      maxDistance: 4.5,
      minPolar: 0.3,
      maxPolar: 1.48,
      minAzimuth: -Math.PI / 4,
      maxAzimuth: Math.PI / 4
    },
    hotspot: { x: 4.90, y: 2.10, z: -20.60, label: 'Portal Entrance' },
    content: {
      tag: '02 / ARCHITECTURE',
      title: 'THE CAFÉ PORTAL',
      description: 'Step into the warm architectural atmosphere of the Columbina space.',
      actionLabel: 'ENTER CAFÉ'
    }
  },
  COUNTER: {
    id: 'COUNTER',
    chapter: '03',
    navLabel: 'COFFEE',
    desktop: {
      pos: new THREE.Vector3(5.60, 3.00, -19.80),
      tgt: new THREE.Vector3(4.90, 1.90, -18.60),
      fov: 40
    },
    mobile: {
      pos: new THREE.Vector3(5.70, 3.10, -20.30),
      tgt: new THREE.Vector3(4.85, 1.85, -18.60),
      fov: 46
    },
    duration: 1300,
    explore: {
      minDistance: 1.0,
      maxDistance: 3.2,
      minPolar: 0.25,
      maxPolar: 1.42,
      minAzimuth: -Math.PI / 3,
      maxAzimuth: Math.PI / 3
    },
    hotspot: { x: 5.00, y: 1.80, z: -18.80, label: 'Order Counter' },
    content: {
      tag: '03 / COFFEE BAR',
      title: 'THE COUNTER',
      description: 'Our central coffee bar showcasing our signature service and menu.',
      actionLabel: 'VIEW OFFERINGS'
    }
  },
  BARISTA: {
    id: 'BARISTA',
    chapter: '04',
    navLabel: 'CRAFT',
    desktop: {
      pos: new THREE.Vector3(3.60, 2.80, -21.40),
      tgt: new THREE.Vector3(4.00, 2.20, -20.60),
      fov: 40
    },
    mobile: {
      pos: new THREE.Vector3(3.50, 2.90, -21.90),
      tgt: new THREE.Vector3(4.00, 2.15, -20.60),
      fov: 45
    },
    duration: 1300,
    explore: {
      minDistance: 0.9,
      maxDistance: 2.8,
      minPolar: 0.3,
      maxPolar: 1.45,
      minAzimuth: -Math.PI / 3,
      maxAzimuth: Math.PI / 3
    },
    hotspot: { x: 4.05, y: 2.10, z: -20.50, label: 'Espresso Bar' },
    content: {
      tag: '04 / CRAFTSMANSHIP',
      title: 'THE BARISTA STATION',
      description: 'Precision brewing and espresso extraction crafted by seasoned baristas.',
      actionLabel: 'DISCOVER CRAFT'
    }
  },
  PASTRY: {
    id: 'PASTRY',
    chapter: '05',
    navLabel: 'PASTRIES',
    desktop: {
      pos: new THREE.Vector3(2.40, 3.00, -19.60),
      tgt: new THREE.Vector3(2.90, 1.70, -19.20),
      fov: 40
    },
    mobile: {
      pos: new THREE.Vector3(2.30, 3.10, -20.00),
      tgt: new THREE.Vector3(2.90, 1.65, -19.20),
      fov: 46
    },
    duration: 1300,
    explore: {
      minDistance: 0.8,
      maxDistance: 2.5,
      minPolar: 0.2,
      maxPolar: 1.40,
      minAzimuth: -Math.PI / 3,
      maxAzimuth: Math.PI / 3
    },
    hotspot: { x: 2.85, y: 1.75, z: -19.25, label: 'Pastry Showcase' },
    content: {
      tag: '05 / ATELIER BAKES',
      title: 'PASTRY DISPLAY',
      description: 'Curated artisanal cakes, viennoiseries, and seasonal baked goods.',
      actionLabel: 'VIEW SELECTION'
    }
  },
  SEATING: {
    id: 'SEATING',
    chapter: '06',
    navLabel: 'ATMOSPHERE',
    desktop: {
      pos: new THREE.Vector3(6.80, 3.10, -18.80),
      tgt: new THREE.Vector3(7.40, 1.20, -18.00),
      fov: 40
    },
    mobile: {
      pos: new THREE.Vector3(6.70, 3.20, -19.40),
      tgt: new THREE.Vector3(7.40, 1.20, -18.00),
      fov: 48
    },
    duration: 1400,
    explore: {
      minDistance: 1.2,
      maxDistance: 4.2,
      minPolar: 0.3,
      maxPolar: 1.45,
      minAzimuth: -Math.PI / 3,
      maxAzimuth: Math.PI / 3
    },
    hotspot: { x: 7.30, y: 1.40, z: -18.10, label: 'Guest Seating' },
    content: {
      tag: '06 / SPACE & COMFORT',
      title: 'LOUNGE SEATING',
      description: 'A relaxed, warm architectural setting designed for slow mornings and quiet meetings.',
      actionLabel: 'EXPERIENCE'
    }
  },
  WIDE: {
    id: 'WIDE',
    chapter: '07',
    navLabel: 'VISIT',
    desktop: {
      pos: new THREE.Vector3(5.20, 4.60, -23.40),
      tgt: new THREE.Vector3(5.40, 2.00, -20.00),
      fov: 42
    },
    mobile: {
      pos: new THREE.Vector3(5.20, 4.80, -24.20),
      tgt: new THREE.Vector3(5.40, 1.90, -20.00),
      fov: 50
    },
    duration: 1600,
    explore: {
      minDistance: 2.0,
      maxDistance: 6.0,
      minPolar: 0.35,
      maxPolar: 1.45,
      minAzimuth: -Math.PI / 4,
      maxAzimuth: Math.PI / 4
    },
    hotspot: { x: 5.35, y: 2.20, z: -20.30, label: 'Full Interior' },
    content: {
      tag: '07 / STOREFRONT & HOURS',
      title: 'VISIT COLUMBINA',
      description: 'Find us in the heart of the city. Doors open every morning.',
      actionLabel: 'HOURS & LOCATION'
    }
  }
};
```

---

## 4. Coherent Experience Flow

### 4.1 Landing Sequence
1. The site loads into **`CINEMATIC`** mode.
2. The initial camera composition is the **`HERO_CUP`** (`01 / THE SIGNATURE OBJECT`).
   - The Columbina cup with its authentic logo sleeve is centrally framed.
   - The depth of field and warm ambient lighting establish the physical café surroundings without overwhelming the focal subject.
3. First user scroll action moves **forward** to **`ENTRY`** (`02 / ARCHITECTURE`), opening up the wider store perspective.
4. Subsequent scrolling advances through:
   `HERO_CUP` → `ENTRY` → `COUNTER` → `BARISTA` → `PASTRY` → `SEATING` → `WIDE` (and backwards seamlessly).

---

## 5. Camera Director & Interpolation Engine

### 5.1 Math & Curves
Transitions use a smooth Quintic Ease-In-Out function:
$$f(t) = t^3(6t^2 - 15t + 10)$$
This provides a soft cinematic acceleration and gentle deceleration into each composition.

- **Look-at Target Tracking:** Linearly interpolates the focal point `tgt` simultaneously with position `pos` to keep subjects centered in the frame throughout camera travel.
- **Vertical FOV Interpolation:** Smoothly lerps between destination FOV values to adapt framing depth.
- **Architectural Parallax (Idle):**
  - While stationary in `CINEMATIC` mode, cursor movement applies a subtle, damped offset:
    - Pitch: $\pm 0.4^\circ$
    - Yaw: $\pm 0.6^\circ$
    - Positional shift: $\le 0.03\text{m}$
  - Re-centers automatically when the pointer leaves the window.

### 5.2 Reduced Motion
When `prefers-reduced-motion: reduce` is active:
- Animated camera flights, idle breathing, and cursor parallax are bypassed.
- Destination changes snap directly or crossfade within 150ms.

---

## 6. Dual-Mode State Machine (`CINEMATIC` vs `EXPLORE`)

```
CINEMATIC MODE (Default)
  │
  ├── Camera driven strictly by CameraDirector
  ├── Subtle mouse parallax active
  ├── Free orbit controls disabled
  ├── Click chapter nav, 3D hotspot, or scroll/swipe → flies to destination
  │
  └── [ ⤢ EXPLORE ] Clicked
        │
        ▼
EXPLORE MODE (Optional)
  │
  ├── OrbitControls enabled around current destination's focal target
  ├── Distance, polar angle, and azimuth angle clamped to destination's specific safe envelope
  ├── Subtle floating [ EXIT EXPLORE ] badge
  │
  └── Nav Tab Clicked / Escape Key / Exit Button
        │
        ▼
Smooth return to authored CINEMATIC shot
```

---

## 7. Performance & Optimization Strategy

1. **Lightweight Hotspot Visibility:**
   - Screen coordinate projection runs on demand during camera movement.
   - Line-of-sight occlusion testing uses throttled raycasting against major architectural meshes (counter and divider walls), running only every 100ms or upon camera flight completion.
2. **GPU & Memory Management:**
   - Retains 100% of the Draco geometry compression benefits (11.17 MB total GLB budget).
   - Pixel ratio clamped to $\min(\text{devicePixelRatio}, 2.0)$ to prevent 4K mobile thermal throttling.
   - Smooth `requestAnimationFrame` loop with idle damping decay.

---

## 8. Verification & Acceptance Criteria

1. **Authored Hero Landing:** Initial load frames the Columbina cup with logo legibility and café ambiance.
2. **No Video Game Flight:** Unrestricted free-roam camera is removed. The default experience is 100% guided and cinematic.
3. **Smooth Flow:** Scrolling forward moves from `HERO_CUP` to `ENTRY` to `COUNTER` without erratic reversing.
4. **Collision & Clipping:** No camera path passes through tables, counter equipment, or exterior walls.
5. **Destination-Specific Explore:** Entering Explore around `HERO_CUP` clamps closely; entering Explore at `SEATING` allows wider atmospheric orbit.
6. **Mobile Usability:** Portrait aspect ratio dynamically adopts authored mobile presets with proper product framing.
7. **Single-File Integrity:** Maintains 100% offline file:// double-click support and GitHub Pages deployment.
