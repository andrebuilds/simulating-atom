# Simulating Atom
### Made by **andrebuilds**

A collection of interactive hydrogen atom physics simulators, written in C++ with OpenGL/GLFW/GLM and Python.

---

## Project Files

### `atom.cpp`
**2D multi-atom simulation with electromagnetic waves**

Displays 20 atoms arranged in a circle that repel each other. Each atom has a proton and an orbiting electron. Left-clicking the mouse emits 25 waves in all directions, which can excite electrons (quantum level jump `n`). When an electron decays back, it emits a wave in return.

**Physics implemented:**
- Circular orbits for discrete energy levels (Bohr model)
- Bohr energy: $E_n = -13.6/n^2$ eV
- Photon absorption and emission for energy transitions
- Inter-atom repulsion force + boundary repulsion

**Where to improve:**
- `Engine` is a **global singleton** — better to pass it as a parameter or use dependency injection
- The `O(N²)` loop for wave-electron collisions is very slow with many atoms/waves: using a **spatial hash grid** would reduce complexity
- Wave energy comparison uses exact float equality (`wave.energy == energyforUp`) — very fragile, should use `abs(a-b) < epsilon`
- `a.pos += a.v` is commented out: atom movement is disabled
- Waves with `energy == 0.0f` are never removed from memory, only skipped: wastes memory over time

---

### `wave_atom_2d.cpp`
**2D single-atom simulation with wavy orbits**

A more artistic/visual version of a single atom. The electron orbit is not circular but **wavy** — the radius oscillates as a sine function, where the number of oscillations is determined by the particle's energy. The user can raise/lower the energy with keys `W/S`, `E/D`, `R/F`.

**Physics implemented:**
- Orbit with modulated radius: $r(\theta) = r_0 + A \cdot \sin(n_{osc} \cdot \theta)$
- Number of oscillations proportional to $-13.6 / E$, echoing quantum numbers

**Where to improve:**
- `numOscillations` is computed twice in both `draw()` and `update()` — should be saved as a struct member
- No fixed delta-time in the loop: simulation speed depends on the monitor FPS
- `amplitude = 50.0f` and `baseOrbit = 200.0f` are hardcoded magic values
- Energy can go positive (ionized particle) without any edge case handling

---

### `atom_realtime.cpp`
**3D real-time quantum orbital viewer with probability current physics**

The most advanced program in the collection. It samples the real hydrogen wave function $\psi_{n,l,m}(r,\theta,\phi)$ using **CDF sampling** for $r$ and $\theta$, distributing N particles in 3D space according to the probability density $|\psi|^2$. Particles are then animated according to the **probability current** $\vec{j} \propto \hbar m / (m_e r \sin\theta)$.

Rendering uses **modern GLSL shaders** (vertex/fragment), VAO/VBO, perspective projection and a mouse-controlled orbital camera.

**Physics implemented:**
- Associated Laguerre polynomials for the radial part
- Associated Legendre polynomials for $\theta$
- Probability current for electron motion
- "Fire" heatmap coloring based on speed/distance

**Where to improve:**
- `sampleR` and `sampleTheta` use `static` variables with `built = false`: **does NOT work if n, l, m are changed at runtime** — the CDF is never rebuilt. Must invalidate it when parameters change
- Sampling runs on a single thread: **parallelizing with `std::async` or a thread pool** would drastically reduce loading times for large N
- All parameters (n, l, m, N) are globals: better to encapsulate them in a config struct
- The shader is hardcoded as a C++ string — better to load it from an external `.glsl` file
- `vec3 vel` in the `Particle` struct does not appear to be used consistently with the declared physics

---

### `atom_raytracer.cpp`
**3D orbital viewer with software ray tracing for spheres**

Similar to `atom_realtime.cpp` for the physics sampling, but uses **ray tracing** to render each particle as a lit sphere. For each screen pixel a ray is cast and sphere intersections are computed, producing physically accurate shading and highlights. Uses an "inferno" colormap to map probability density to color.

**Physics implemented:**
- Same quantum sampling as `atom_realtime.cpp`
- Ray-sphere intersection for every particle
- Sphere normal shading with a point light source

**Where to improve:**
- Ray tracing is **CPU-bound and O(pixels × particles)**: with 100,000 particles and 800×600 pixels it is impractical without optimization. A **BVH (Bounding Volume Hierarchy)** would reduce complexity from O(N) to O(log N) per ray
- The render loop is single-threaded: **splitting pixel rows across threads** with `std::thread` or OpenMP would give linear speedup with core count
- Same static CDF bug as in `atom_realtime.cpp`
- `LightingScaler = 700` is a hardcoded magic value

---

### `schrodinger.py`
**Python generator for complex wave function samples**

Python script that generates samples of the complex wave function $\psi_{n,l,m}$ using rejection sampling, and saves them as JSON in the `orbitals/` folder. Samples include the 3D position and the complex $\psi$ value (real and imaginary parts). The generated JSON can be loaded into the C++ viewers.

**Where to improve:**
- Uses **rejection sampling** (very inefficient for high n states): better replaced with CDF sampling as in the C++ files
- `max_prob` is updated on-the-fly during sampling: this introduces a statistical bias since early samples use an underestimated threshold. Better to do a pre-pass to find `max_prob`
- No progress bar for large N: add `tqdm`
- JSON files can become very large (hundreds of MB for N=100000): consider a binary format like `numpy .npy`

---

### `mesh.json`
JSON file containing a 3D mesh (vertices), likely generated by `schrodinger.py` or an external tool, used as input by one of the C++ viewers.

---

## Dependencies

| Library | Purpose |
|---|---|
| **GLEW** | OpenGL extension loading |
| **GLFW3** | Window creation and input handling |
| **GLM** | Vector/matrix math |
| **Python 3** + numpy, scipy | Only for `schrodinger.py` |

## Building (Windows with vcpkg)

```cmd
# Install dependencies
C:\vcpkg\vcpkg install glew glfw3 glm

# Build from VS Code command palette
Ctrl+Shift+P → Run Task → choose the file to compile
```

## Controls

| Program | Key / Action | Effect |
|---|---|---|
| `atom.exe` | Left click | Emits 25 photon waves |
| `wave_atom_2d.exe` | W / S | Increase / decrease energy (×0.01) |
| `wave_atom_2d.exe` | E / D | Increase / decrease energy (×0.1) |
| `wave_atom_2d.exe` | R / F | Increase / decrease energy (×1.0) |
| `atom_realtime.exe` | Mouse drag | Rotate orbital camera |
| `atom_raytracer.exe` | Mouse drag | Rotate orbital camera |

---

*Made by **andrebuilds***
