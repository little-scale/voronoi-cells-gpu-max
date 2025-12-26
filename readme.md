# Voronoi Cell Simulator

A GPU-accelerated 2D Voronoi cell simulator with real-time physics, mouse interaction, OSC output, and Max/MSP integration. Designed for audio-visual performance and sonification.

## Overview

This simulator renders animated Voronoi cells using WebGL2 fragment shaders. Cells move according to configurable physics (turbulence, attraction, separation forces) and can be manipulated in real-time via mouse interaction. The system outputs comprehensive analysis data via OSC and Max API for sonification.

## Features

- **GPU-accelerated rendering** - Voronoi distance calculations run entirely on the GPU
- **Real-time physics** - Turbulence, attraction to center, separation forces, velocity damping
- **Mouse interaction** - Attract or repel cells with click and drag
- **OSC output** - WebSocket-based OSC for integration with any OSC-capable software
- **Max/MSP integration** - Native jweb bindings for Max/MSP
- **Comprehensive analysis** - Cell areas, nearest neighbor distances, collision detection, spatial spread

---

## Quick Start

1. Open `voronoi.html` in a modern browser (Chrome/Firefox/Edge)
2. Cells will animate automatically
3. Click and drag to attract cells toward mouse
4. Shift + drag to repel cells
5. Press `H` to hide sidebar for fullscreen view

---

## Controls

### Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `H` | Toggle sidebar visibility |
| `I` | Toggle info overlay |
| `Space` | Pause/Resume simulation |
| `R` | Reset cells to random positions |

### Mouse Interaction

| Action | Effect |
|--------|--------|
| Click + Drag | Attract cells toward mouse |
| Shift + Click + Drag | Repel cells away from mouse |

### Sidebar Controls

#### Simulation
- **Cell Count** (2-64) - Number of Voronoi cells
- **Speed** (0-1) - Simulation speed multiplier
- **Turbulence** (0-1) - Random noise-based movement
- **Attraction** (-1 to 1) - Force toward/away from center
- **Pause/Reset** - Control simulation state

#### Appearance
- **Border Width** (0-10) - Thickness of cell borders
- **Color A / Color B** - Gradient colors for cells (interpolated by cell index)
- **Background** - Background color
- **Show Centroids** - Display cell center points

---

## Data Output

The simulator provides rich data for sonification and visualization.

### Per-Cell Data

| Property | Range | Description |
|----------|-------|-------------|
| `x`, `y` | 0-1 | Cell position (normalized) |
| `vx`, `vy` | ~±0.01 | Cell velocity |
| `speed` | 0+ | Velocity magnitude |
| `nearestDist` | 0-1 | Distance to nearest neighbor cell |
| `nearestNeighbor` | 0-N | Index of nearest cell |
| `density` | 1-100+ | Local density (1/nearestDist) |
| `area` | 0-1 | Approximate Voronoi cell area (proportion of total) |

### Aggregate Data

| Property | Description |
|----------|-------------|
| `avgPos` | Center of mass (x, y) |
| `avgVel` | Average cell speed |
| `spread` | Spatial distribution (standard deviation) |
| `avgDensity` | Average local density |
| `minNearestDist` | Distance of closest cell pair |
| `maxNearestDist` | Distance of most isolated cell |
| `collisionCount` | Number of collision events this frame |

### Events

| Event | Data | Description |
|-------|------|-------------|
| `collision` | cellA, cellB, distance | Fires when two cells enter collision threshold |
| `cellCreated` | index, x, y | Fires when a new cell is added |
| `cellDestroyed` | index, x, y | Fires when a cell is removed |

---

## OSC Integration

### Setup

1. Run a WebSocket-to-OSC bridge (e.g., `osc-web`, custom Node.js server)
2. Enter the WebSocket URL in the sidebar (default: `ws://localhost:8080`)
3. Click "Connect"

### OSC Address Format

All addresses are prefixed with `/voronoi`

#### Per-Cell Messages (throttled to every ~3 frames)

```
/voronoi/cell/0/pos        [x] [y]
/voronoi/cell/0/vel        [vx] [vy]
/voronoi/cell/0/speed      [speed]
/voronoi/cell/0/nearestDist    [distance]
/voronoi/cell/0/nearestNeighbor [index]
/voronoi/cell/0/density    [density]
/voronoi/cell/0/area       [area]
```

#### Aggregate Messages (every frame)

```
/voronoi/cells/count       [count]
/voronoi/aggregate/avgPos  [x] [y]
/voronoi/aggregate/avgVel  [velocity]
/voronoi/aggregate/spread  [spread]
/voronoi/aggregate/avgDensity  [density]
/voronoi/aggregate/minNearestDist  [distance]
/voronoi/aggregate/maxNearestDist  [distance]
/voronoi/aggregate/collisionCount  [count]
```

#### Event Messages (immediate)

```
/voronoi/event/collision      [cellA] [cellB] [distance]
/voronoi/event/cellCreated    [index] [x] [y]
/voronoi/event/cellDestroyed  [index] [x] [y]
```

#### Incoming OSC Control

```
/voronoi/cells/count   [n]        Set cell count
/voronoi/speed         [0-1]      Set speed
/voronoi/turbulence    [0-1]      Set turbulence
/voronoi/reset                    Reset simulation
```

---

## Max/MSP Integration

### Setup

1. Create a `jweb` object in Max
2. Load `voronoi.html`
3. Connect inlets and outlets

### Inlet Messages

#### Control

| Message | Arguments | Description |
|---------|-----------|-------------|
| `setCellCount` | int (2-64) | Set number of cells |
| `setSpeed` | float (0-1) | Set simulation speed |
| `setTurbulence` | float (0-1) | Set turbulence amount |
| `setAttraction` | float (-1 to 1) | Set center attraction |
| `setBorderWidth` | float (0-10) | Set border width |
| `setCollisionThreshold` | float (0.01-0.5) | Set collision detection distance |
| `reset` | | Reset cells to random positions |
| `pause` | | Pause simulation |
| `resume` | | Resume simulation |
| `toggle` | | Toggle pause state |

#### Data Requests

| Message | Arguments | Description |
|---------|-----------|-------------|
| `getCellData` | | Request cell data (outputs JSON on `cellData` outlet) |
| `getAnalysis` | | Request full analysis (outputs JSON on `analysis` outlet) |
| `enableOutput` | enable (0/1), intervalMs | Enable/disable continuous output |

### Outlet Messages

#### Startup

```
ready 1
```

Sent when jweb is loaded and ready.

#### Continuous Output (when enabled)

**Aggregate data:**
```
aggregate [count] [avgX] [avgY] [avgVel] [spread] [minDist] [maxDist]
```

**Per-cell data (one message per cell):**
```
cell [index] [x] [y] [vx] [vy] [speed] [nearestDist] [density] [area]
```

**Collision events:**
```
collision [cellA] [cellB] [distance]
```

**Cell lifecycle events:**
```
cellCreated [index] [x] [y]
cellDestroyed [index] [x] [y]
```

#### On-Demand Data

```
cellData [JSON string]
analysis [JSON string]
```

### Example Max Patch

```
[jweb voronoi.html @width 800 @height 600]
    |
[route ready aggregate cell collision cellData analysis]
    |         |        |       |          |        |
[print]  [unpack]  [unpack] [print]   [dict]   [dict]
```

To start continuous output at 20fps:
```
[message enableOutput 1 50]
    |
[jweb]
```

---

## JavaScript API

Access via `window.voronoi` in browser console or jweb.

### Control Methods

```javascript
window.voronoi.setCellCount(16)      // Set cell count (2-64)
window.voronoi.setSpeed(0.5)         // Set speed (0-1)
window.voronoi.setTurbulence(0.3)    // Set turbulence (0-1)
window.voronoi.setAttraction(0.1)    // Set attraction (-1 to 1)
window.voronoi.setBorderWidth(1.0)   // Set border width (0-10)
window.voronoi.setColorA([1,0,0])    // Set color A [r,g,b] (0-1)
window.voronoi.setColorB([0,1,1])    // Set color B [r,g,b] (0-1)
window.voronoi.setCollisionThreshold(0.08)  // Set collision distance

window.voronoi.reset()               // Reset cells
window.voronoi.pause()               // Pause simulation
window.voronoi.resume()              // Resume simulation
window.voronoi.toggle()              // Toggle pause state

window.voronoi.toggleSidebar()       // Toggle sidebar visibility
window.voronoi.toggleSidebar(false)  // Hide sidebar
window.voronoi.toggleInfo()          // Toggle info overlay
```

### Event Callbacks

```javascript
// Register callback for when a cell is created
window.voronoi.onCellCreated((index, x, y) => {
    console.log(`Cell ${index} created at (${x}, ${y})`);
});

// Register callback for when a cell is destroyed
window.voronoi.onCellDestroyed((index, x, y) => {
    console.log(`Cell ${index} destroyed from (${x}, ${y})`);
});
```

### Data Methods

```javascript
// Get current state
window.voronoi.getState()
// Returns: { numCells, speed, turbulence, attraction, borderWidth, 
//            colorA, colorB, bgColor, showCentroids, paused, time }

// Get cell data with analysis
window.voronoi.getCellData()
// Returns: [{ index, x, y, vx, vy, speed, nearestDist, 
//             nearestNeighbor, density, area }, ...]

// Get full analysis
window.voronoi.getAnalysis()
// Returns: {
//   cells: [{ index, x, y, vx, vy, speed, nearestDist, 
//             nearestNeighbor, density, area }, ...],
//   aggregate: { avgPos: {x,y}, avgVel, avgDensity, spread,
//                minNearestDist, maxNearestDist }
// }

// Get/set individual cell position
window.voronoi.getCellPosition(0)    // Returns: { x, y }
window.voronoi.setCellPosition(0, 0.5, 0.5)  // Set cell 0 to center
```

---

## Sonification Ideas

### Direct Mapping (per cell → voice)

- Each cell = oscillator or synth voice
- `x` → pitch (left=low, right=high)
- `y` → filter cutoff or timbre
- `speed` → amplitude or vibrato depth
- Works well with 8-16 cells as polyphonic voices

### Spatial Audio

- `x` → stereo pan
- `y` → reverb amount or delay time
- Creates a spatial field that moves with cells

### Edge Tension

- `nearestDist` = musical tension
- Small values → dissonance, density, activity
- Large values → consonance, space, calm
- `minNearestDist` (aggregate) → global tension parameter

### Cell Area

- `area` → note duration or pitch
- Larger cells = lower pitch or longer sustain
- Smaller cells = higher pitch or shorter notes

### Collision Events

- Trigger percussive sounds on `/event/collision`
- Use `distance` to control velocity/intensity
- Cell indices can select different sounds

### Aggregate Dynamics

- `spread` → mix between dense/sparse textures
- `avgVel` → global energy/activity
- `avgPos` → melodic center or drone pitch

### Mouse as Performer

- Attract creates compression → crescendo, harmonic density
- Repel creates expansion → decrescendo, sparse texture
- Direct physical metaphor for musical dynamics

---

## Technical Details

### Physics Simulation

Each frame, cells are updated with:

1. **Separation forces** - Cells repel when closer than `minSeparation` (0.06)
2. **Turbulence** - Sine/cosine noise field based on position and time
3. **Center attraction** - Force toward (0.5, 0.5) scaled by `attraction`
4. **Velocity integration** - Position += velocity × speed
5. **Damping** - Velocity *= 0.98
6. **Boundary wrapping** - Toroidal topology (cells wrap at edges)

### Voronoi Rendering

The fragment shader calculates for each pixel:
1. Distance to all cell positions
2. Find closest and second-closest cells
3. Edge = difference between closest and second-closest distances
4. Color interpolation based on cell index
5. Border rendering via smoothstep on edge distance

### Cell Area Estimation

Uses Monte Carlo sampling:
1. Generate N random points (default 300)
2. For each point, find closest cell
3. Count points per cell
4. Normalize to get area proportion

This is approximate but sufficient for sonification purposes.

### Performance Notes

- Tested with up to 64 cells at 60fps
- Cell area calculation is the most expensive operation
- OSC output is throttled to reduce bandwidth
- GPU handles all rendering; CPU handles physics and analysis

---

## Browser Compatibility

- **Chrome** - Recommended, best performance
- **Firefox** - Fully supported
- **Edge** - Fully supported
- **Safari** - WebGL2 required (Safari 15+)

Requires WebGL2 support.

---

## Files

- `voronoi.html` - Single-file application (HTML + CSS + JavaScript + GLSL)

---

## License

MIT License - Free for personal and commercial use.

---

## Version History

- **1.0** - Initial release with 2D Voronoi, physics, OSC, Max API, and analysis features
