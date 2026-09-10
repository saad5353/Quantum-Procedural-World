# Quantum Randomness → Procedural World

## Overview

This project generates a 2D island world where **quantum measurement outcomes**
control the **parameters** of a classical procedural generation algorithm.
The terrain itself is produced with a standard layered/fractal noise technique;
quantum randomness only decides *which* world (out of many possible parameter
combinations) gets built — similar to how a "world seed" works in games, except
the seed bits come from a simulated quantum circuit instead of a classical PRNG.

Quantum measurement and classical procedural generation are kept strictly
separate throughout this project:

- **Quantum layer**: Hadamard + measurement circuit → raw random bits.
- **Classical layer**: fractal noise, island masking, thresholding, biome
  classification, river tracing, road pathfinding, structure placement, and
  rendering — all standard, deterministic procedural generation algorithms
  driven by the quantum-derived parameters.

The final world includes:

- A quantum-seeded island heightmap
- 10 biome types (deep water, shallow water, beach, grass, forest, mountain,
  snow, desert, tundra, swamp)
- Villages placed under classical placement constraints
- Rivers traced from quantum-chosen mountain sources down to the sea
- Roads connecting villages via pathfinding across the terrain

![Final world overview] <img width="966" height="989" alt="world 5" src="https://github.com/user-attachments/assets/5f35647b-fc68-4b19-829c-37bdd2dd1950" />


*Caption: The complete generated world — biomes, rivers (blue), roads (dashed brown), and villages (red triangles).*

---

## Algorithm

### Core Pipeline

1. Build a quantum circuit: put `N` qubits into superposition with Hadamard
   gates, then measure all qubits.
2. Execute the circuit for many shots on the Qiskit Aer simulator, retrieving
   each shot's bitstring individually (`memory=True`) and concatenating them
   into one long quantum-measured bitstream.
3. Feed the bitstream through a cursor-based `QuantumBitStream` reader that
   converts consecutive bit groups into integers/floats mapped to specific
   ranges — these become the **world parameters**.
4. The same bitstream drives a quantum-informed Fisher–Yates shuffle that
   builds the permutation table used by the noise function.
5. Generate a base heightmap with layered (fractal/octave) value noise using
   the quantum-derived scale, octave count, persistence, and lacunarity.
6. Apply a radial island mask (quantum-derived falloff exponent) and a
   mountain-shaping exponent (quantum-derived intensity).
7. Threshold the continuous heightmap into biomes using quantum-influenced
   thresholds.
8. Place village structures at quantum-derived coordinates, filtered by
   classical constraints.
9. Trace rivers from quantum-chosen mountain sources down to sea level.
10. Connect villages with roads using classical pathfinding.
11. Validate constraints and render the result with matplotlib.

---

## Quantum Parameter Generation

- **Circuit**: `H` gates on every qubit + measurement (no entanglement, no
  gate optimization — deliberately simple so the randomness source is
  transparent).
- **Simulator**: Qiskit Aer (`AerSimulator`), run with `memory=True` so every
  individual shot's bitstring is retrievable (not just aggregated counts).
- All shot bitstrings are concatenated into a single bit pool.
- A `QuantumBitStream` class reads fixed-size bit chunks off a cursor and
  converts them into integers, normalized floats, and range-mapped floats.
- **No `random` or `numpy.random` calls are used as an entropy source
  anywhere in the parameter-derivation, shuffle, or placement logic.**
  (NumPy is used only for vectorized array *math*, not randomness.)

### Core Quantum-Derived Parameters

| Parameter | Controls |
|---|---|
| `island_falloff` | Island shape (how sharply land fades to ocean at edges) |
| `terrain_scale` | Noise feature size / terrain roughness |
| `octaves`, `persistence`, `lacunarity` | Noise layering detail |
| `mountain_intensity` | Peak sharpness/flatness |
| `water_level` | Sea level |
| `forest_density` | Proportion of grass converted to forest |
| `village_count` | Number of villages to attempt to place |
| `snow_threshold_offset`, `mountain_threshold_offset` | Biome boundary tuning |
| (bitstream also drives) | Noise permutation table shuffle, village candidate coordinates |

### Extended Quantum-Derived Parameters (Bonus Biomes & Rivers)

| Parameter | Controls |
|---|---|
| `moisture_scale` | Feature size of the moisture noise field |
| `equator_shift` | Position of the "warm equator" band in the temperature map |
| `desert_moisture_t`, `desert_temp_t` | Desert classification thresholds |
| `swamp_moisture_t`, `swamp_height_t` | Swamp classification thresholds |
| `tundra_temp_t`, `tundra_moisture_t` | Tundra classification thresholds |
| `forest_moisture_t` | Forest classification threshold (extended model) |
| `river_count` | Number of river sources to attempt to place |
| (bitstream also drives) | Moisture noise permutation shuffle, river source coordinates |

---

## Terrain Generation

- A vectorized value/Perlin-style noise function (`perlin_vectorized`) is
  implemented from scratch using a gradient table and a quantum-shuffled
  permutation table.
- Multiple octaves of this noise are summed (fractal Brownian motion) to
  produce a natural-looking base heightmap.
- A radial mask multiplies the heightmap to create an island silhouette.
- An exponent reshapes the heightmap to control mountain sharpness.
- The final heightmap is normalized to `[0, 1]`.

![Heightmap and biome map] <img width="1589" height="803" alt="world 1" src="https://github.com/user-attachments/assets/df520a4f-4ecc-44b2-bac1-133d90f59861" />

*Caption: Raw quantum-seeded fractal heightmap (left) next to the classified biome map (right).*

---

## Biome Generation

### Base Biomes (Height-Only)

Height values are thresholded into 7 biome types:

`deep water`, `shallow water`, `beach`, `grass`, `forest`, `mountain`, `snow`

Thresholds are computed from the quantum-derived `water_level` and small
quantum-derived offsets for the mountain/snow boundaries. A secondary noise
layer (same permutation table, different frequency) scatters forest patches
within grass areas, gated by the quantum-derived `forest_density`.

### Extended Biomes (Height + Moisture + Temperature)

To add richer variety without breaking coherence, two additional classical
fields are introduced:

- **Moisture map** — a second fractal noise layer, built from an
  independently quantum-shuffled permutation table.
- **Temperature map** — a latitude-style gradient (warm near a
  quantum-shifted "equator row", cold near the poles, further cooled by
  elevation).

Biome is decided by **height + moisture + temperature together**, following
a Whittaker-diagram style approach. This produces 3 additional biomes:

`desert` (warm + dry), `tundra` (cold + dry), `swamp` (low + wet)

for a total of 10 biome types. Quantum measurement only sets the
*thresholds* and the *equator position/noise scale* — never individual
pixels — so biomes remain geographically sensible and smooth.

![Moisture and temperature maps] <img width="1041" height="490" alt="world 3" src="https://github.com/user-attachments/assets/76d0d32c-a903-4e20-a74d-3ca393f7cb8c" />
*Caption: Quantum-seeded moisture map (left) and quantum-shifted temperature gradient (right), used to classify extended biomes.*
```
```
![Extended biome map] <img width="558" height="581" alt="wordl 4" src="https://github.com/user-attachments/assets/ffe59aa8-71d1-4b21-a44d-3388d96dacbb" />

*Caption: Final 10-biome map including desert, tundra, and swamp regions.*

---

## Rivers

Rivers are generated with a classical **steepest-descent walk**:

1. A handful of high-elevation "source" points are chosen (quantum-selected
   from the highest terrain cells, spaced apart via `river_count` and
   minimum-spacing constraints).
2. From each source, the algorithm repeatedly steps to the lowest
   neighboring cell (8-connected) until sea level is reached, or the path
   gets stuck in a local pit (no downhill neighbor).

This is a standard hydraulic flow-tracing technique used in procedural
terrain generation. It is entirely deterministic classical logic — quantum
randomness only decides **where each river begins**.

![Rivers traced on terrain] <img width="966" height="989" alt="world 5" src="https://github.com/user-attachments/assets/c7a37af2-c2e8-43b7-978f-b302f7f929bc" />
*Caption: Rivers (blue) traced via steepest-descent from quantum-chosen mountain sources down to the sea.*

---

## Roads

Villages are connected using classical pathfinding over a **walkable cost
grid** derived from the biome map:

| Biome | Relative Cost |
|---|---|
| Grass | Low |
| Forest | Low–medium |
| Beach | Low–medium |
| Desert | Medium |
| Tundra | Medium |
| Swamp | High |
| Mountain | Very high |
| Snow | Very high |
| Water | Impassable |

To avoid an expensive all-pairs shortest-path search:

1. Villages are first connected using a **minimum spanning tree** (Prim's
   algorithm) based on straight-line distance.
2. Only the resulting MST edges are pathfound with **A\*** across the
   biome-weighted cost grid, producing terrain-aware road paths that avoid
   water and prefer easy terrain.

Quantum randomness has no role in this stage — it is purely classical
constraint-based generation layered on top of the quantum-seeded map.

---

## Constraints

All structure placement respects classical constraints, validated after
generation:

- Villages must land on **grass or forest** biome (not water/beach/mountain/
  snow/desert/tundra/swamp, depending on configuration).
- Villages must be **inside the world grid** (guaranteed by coordinate
  wrapping).
- Villages must be **at least `MIN_VILLAGE_DISTANCE` cells apart** from each
  other.
- Roads must avoid impassable terrain (water) entirely, enforced by the A*
  cost grid (`inf` cost cells are never traversed).
- Rivers terminate once they reach sea level or a local pit, preventing
  infinite or nonsensical paths.

If a quantum-derived candidate position violates a constraint (e.g. a
village lands in water), it is discarded and the next candidate is drawn
from the bitstream (rejection sampling), up to a maximum attempt budget.

---

## Installation

```bash
pip install qiskit qiskit-aer
