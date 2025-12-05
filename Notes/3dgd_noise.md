# 3D Game Development — Unreal Materials
## Noise (Blue, Perlin/Gradient, Simplex, Voronoi/Worley, Brownian/fBm)

> This lesson explores the main noise families used in real-time shading with Unreal Engine materials. We’ll explain how each noise is **generated**, what it **looks** like, where it’s **useful**, and what it **costs** on the GPU. We’ll also cover practical Unreal patterns, when to **pre-bake** noise to textures, and how to layer noise with **fBm** and **domain warping**.

## Learning outcomes

By the end of this lesson, you should be able to:

- Describe how **Blue**, **Perlin/Gradient**, **Simplex**, **Voronoi/Worley**, and **Brownian/fBm** noise are constructed.
- Select an appropriate noise for common material tasks (mask breakup, roughness variation, emissive modulation, UV flow).
- Implement practical versions in **Unreal Materials** using texture samples, material functions, and the Vector/Noise nodes.
- Optimize: decide when to swap procedural nodes for **pre-baked textures**, and control octave counts and sampler use.

## Orienting to noise in Unreal

You’ll generally use noise in one of three ways:

1. **Texture-based** noise — import tileable noise textures and sample them. This is fast, stable, and cross-platform predictable.
2. **Procedural nodes** — use **Noise** (scalar) or **Vector Noise** to compute values on the fly. Flexible, but heavier on ALU.
3. **Material Functions** — wrap either approach so you can reuse consistent parameters (frequency, amplitude, thresholds) via Material Instances.

A solid workflow is: **prototype procedurally → pre-bake the look to textures → keep controls** (tiling, intensity) so artists can still tune.

## Blue noise — “random, but nicely spread out”

### Intuition
White noise clumps; blue noise **suppresses low-frequency clumping**, giving randomness that feels evenly distributed. This makes dithering and thresholding look cleaner to the eye.

### How it’s produced
- **Poisson-disk sampling (“dart throwing”)**  
  1) Place a random point.  
  2) Reject any new point closer than a radius **r** to existing points.  
  3) Repeat until saturated.  
  Bake the point set into an image (sometimes as a rank/ordering so thresholds pick well-spaced pixels).
- **Bridson’s algorithm** speeds this up with an “active list” of candidates.
- Many tools precompute textures whose **power spectrum** concentrates energy at higher frequencies (hence “blue”).

### What ends up in the texture?
A single-channel image whose values form a blue-noise distribution. Sample it in UV or world space. For subtle animation, nudge UVs slightly over time.

### Why it’s useful
Best-in-class for **dithering** (opacity, lighting thresholds) and breaking up banding. Great for **emissive sparkle** or roughness variation without blotches.

### Cost
**Very cheap** as a texture sample (1 sampler).

## Perlin (Gradient) noise — “smooth variation from hidden gradients”

### Intuition
Lay down a **grid**. At each grid corner store a **random unit gradient vector** (a direction). A point inside a cell measures how aligned it is with each corner’s gradient (dot product), then **smoothly blends** those contributions.

### How it’s produced (2D)
1) Cell & local coords:  
   `i=floor(x), j=floor(y); u=x−i, v=y−j`.
2) Corner gradients via a repeatable hash: `g00, g10, g01, g11`.
3) Corner contributions (dot products):  
   `n00=dot(g00,(u, v))`, `n10=dot(g10,(u−1, v))`,  
   `n01=dot(g01,(u, v−1))`, `n11=dot(g11,(u−1, v−1))`.
4) Smooth fade: `fade(t)=t²(3−2t)`; `fu=fade(u)`, `fv=fade(v)`.
5) Interpolate:  
   `nx0 = lerp(n00, n10, fu)`  
   `nx1 = lerp(n01, n11, fu)`  
   `value = lerp(nx0, nx1, fv)` (≈ `[-1,1]`).

### Why it’s useful
**Cloudy** continuous patterns for terrain masks, fog, albedo/roughness breakup. A default “natural” scalar noise.

### Cost
Procedural **moderate**; **texture-baked** Perlin is **cheap**. Layer multiple octaves for richness (see fBm).

## Simplex noise — “Perlin’s smoother cousin on triangles/tetrahedra”

### Intuition
Perlin’s axis-aligned grid can leave subtle streaks, especially in 3D. **Simplex** reshapes space so evaluation happens on **simplices** (triangles in 2D, tetrahedra in 3D). Fewer corners contribute per sample; interpolation is more isotropic.

### How it’s produced (2D sketch)
1) **Skew** input so squares become equilateral triangles:  
   `s=(x+y)*F2`, with `F2=(√3−1)/2`;  
   `i=floor(x+s), j=floor(y+s)`.
2) **Unskew** back to original space:  
   `t=(i+j)*G2`, `G2=(3−√3)/6`;  
   `X0=x−(i−t)`, `Y0=y−(j−t)`.
3) Decide which triangle of the simplex we’re in (`X0>Y0` test).
4) For the 3 corners, derive hashed gradients and contributions:  
   `tk = max(0, 0.5 − xk² − yk²)`;  
   `contrib_k = (tk^4) * dot(grad_k, (xk, yk))`.
5) Sum scaled contributions.

### Why it’s useful
Similar look to Perlin but with **fewer artifacts**, especially in 3D/volumetrics (fog, smoke).

### Cost
Procedural **lighter than Perlin** per dimension, but still heavier than a texture sample. Prefer **pre-baked** Simplex textures for surface materials.

## Voronoi / Worley (Cellular) — “closest-seed distance fields”

### Intuition
Scatter **seed points**; for each position, measure the distance to the **nearest** (F1) and optionally **second nearest** (F2) seed. The distance field forms **cells** and **ridges** like cracked mud, scales, or grout.

### How it’s produced (2D)
1) Partition space into integer cells; each has a hash-placed seed.
2) For a query `(x,y)`, check the local cell and neighbors (e.g., 3×3).
3) Compute distances `d_k` to each candidate seed.
4) Track `F1 = min(d_k)`, `F2 = second_min(d_k)`.
5) Output options: **F1**, **F2**, or **F2−F1** (strong borders).

### Turning it into art
Apply **SmoothStep** to carve crisp cell interiors or thin ridges. `F2−F1` is excellent for **cracks**.

### Why it’s useful
Cellular patterns for **stone, pores, scales, grout**, chipped paint.

### Cost
Procedural is **heavy** (many distance checks). **Texture-baked Voronoi** is far cheaper and predictable.

## Brownian / fractal Brownian motion (fBm) — “layers of detail at many scales”

> fBm is **not a new noise**; it’s a **stacking strategy** applied to any base noise (Perlin/Simplex/Value). The idea is to sum multiple **octaves** of the base noise: each octave has **higher frequency** (more detail) and **lower amplitude** (less intensity).

### How it’s produced
- Choose octaves **O**, lacunarity **L** (frequency multiplier, typically 2.0), and gain **G** (amplitude multiplier, typically 0.5).
- Sum:
  ```
  sum=0; amp=1; freq=1; norm=0
  repeat k in 0..O-1:
      sum  += amp * Noise(x*freq, y*freq)
      norm += amp
      amp  *= G
      freq *= L
  value = sum / norm
  ```
- Variants: **turbulence** uses `abs(noise)` for ridged looks; **domain warping** distorts coordinates before sampling (see below).

### Why it’s useful
One pattern with **broad forms and fine detail**—great for rocks, wood, clouds, water, and any surface that needs scale richness.

### Cost
Roughly linear in **octave count**. Using **textures** for the base noise keeps it efficient.

## Domain warping (bonus technique)

Instead of sampling at plain UVs, first **distort the coordinates** with another noise (often a vector field). This kills repetition and produces **swirls/marbling**:

```
V  = VectorNoise(UV * warpFreq)        // float3
UV' = UV + V.xy * warpAmount
N   = Sample(T_Perlin, UV' * baseFreq) // richer, less repetitive
```

Cost increases with extra samples; use modest warp amounts to avoid stretching.

## Pre-baked noise textures (why they matter)

- **Performance & predictability.** Texture fetches are fast and behave the same across platforms; procedural noise can vary by shader model or be ALU-expensive.
- **Art control.** You can ensure tiling, pack multiple variants (e.g., F1/F2/F2−F1 in RGB), and normalize ranges.
- **Stability.** Proper mips reduce shimmer; UV/world-space anchoring behaves well with TAA.
- **Sampler budget.** Prefer **Sampler Source: Shared Wrap/Clamp** and reuse the same noise textures across materials.

**Workflow:** explore with procedural nodes → bake to textures (tileable) → keep parameters (frequency, intensity, thresholds) exposed via Material Instances.

## Practical Unreal patterns

### Emissive breakup with blue noise
```text
BN     = Sample(T_BlueNoise, UV * 6)
Pulse  = 0.5 + 0.5 * Sin(Time * Speed)
Glow   = (Base + Amp * Pulse) * lerp(0.9, 1.1, BN)
Emissive = GlowColor * Glow
```

### Roughness variation with Perlin fBm (3 octaves)
```text
P1 = Sample(T_Perlin, UV * 1.5)
P2 = Sample(T_Perlin, UV * 3.0) * 0.5
P3 = Sample(T_Perlin, UV * 6.0) * 0.25
fBm = (P1 + P2 + P3) * (1.0 / 1.75)
Roughness = Remap(fBm, 0..1 → 0.25..0.8)
```

### UV distortion with vector noise
```text
V   = VectorNoise(UV * 2.0)          // float3
UV2 = UV + V.xy * 0.02 * WarpAmount
Albedo  = Sample(T_Albedo,  UV2)
Normals = Sample(T_Normals, UV2)
```

### Cellular stone mask (Voronoi)
```text
Cell  = Sample(T_Voronoi, UV * 4)
Edges = SmoothStep(0.4, 0.6, Cell)
Albedo = lerp(GroutColor, StoneColor, Edges)
```

## Comparison table

| Noise | Output | How it’s generated (summary) | Visual character | Typical uses | Tileable? | Stability (mips/TAA) | Typical Unreal approach | Relative cost* |
|---|---|---|---|---|---|---|---|---|
| **Blue** | Scalar | Poisson-disk/optimized distributions; encode ranks/values in a texture | Evenly spread speckle, no blotches | Dithered transparency, mask breakup, emissive sparkle | Yes | Very high | **Texture sample** | ★ |
| **Perlin (Gradient)** | Scalar | Grid of random gradients; dot & smooth fade interpolation | Smooth, cloud-like | Terrain, fog, roughness/albedo breakup | Yes (variant) | High | Texture (or procedural Noise node) | ★★–★★★ (tex ≈ ★★) |
| **Simplex** | Scalar | Skew to simplex lattice; 3 (2D) or 4 (3D) corners with tapered contributions | Perlin-like, fewer artifacts, good in 3D | Volumetrics, continuous masks | Yes (variant) | High | Texture or Custom node | ★★–★★★ (tex ≈ ★★) |
| **Voronoi/Worley** | Scalar | Distance to nearest/2nd-nearest hashed seeds | Cellular cells & ridges | Stone, pores, scales, grout, cracks | Yes (pre-baked) | High | **Texture** (packed variants) | ★★★–★★★★ (tex ≈ ★★) |
| **Brownian / fBm** | Scalar | Sum octaves of a base noise with ↑freq & ↓amp | Multi-scale, natural richness | Rocks, wood, clouds, water | If base is | High | Multi-sample textures | ★★–★★★★ (per octave) |
| **Vector Noise** | Vector3 | Procedural vector field (direction+magnitude) | Directional flow/warp | UV distortion, wavy normals | Varies | High | Vector Noise node or RGB noise | ★★★ |

\* Relative cost is approximate: ★ (cheapest) → ★★★★★ (heaviest). Texture sampling is almost always cheaper than equivalent procedural ALU.

## Choosing the right noise (quick guidance)

- **Dithering & thresholds (TAA-friendly):** **Blue noise**.  
- **Organic masks & roughness variation:** **Perlin/Simplex**, often as **fBm** (3–5 octaves).  
- **Cellular looks (stone, scales, pores, grout):** **Voronoi/Worley** with **SmoothStep** thresholds.  
- **UV flow/heat haze/water wobble:** **Vector Noise** (or RGB noise as a flow field).  
- **Kill repetition:** add **domain warping** with modest warp amounts.

## Reflective questions

1. Your translucent material bands under TAA. Why does **blue noise** help, and how would you sample it (space, scale, animation) to stay stable?  
2. You need a convincing **lava flow** with directional movement: which base noise and techniques (vector noise, fBm, domain warping) would you combine, and how would you keep the sampler/ALU cost in check?  
3. A prototype uses the **procedural Noise** node and stutters on target hardware. Outline a concrete bake plan: which texture(s) you’d author, how you’d ensure tiling/mips, and which parameters you’d keep in the Material Instance.  
4. A hero rock using single-octave Perlin looks “blobby” up close. Describe an **fBm** stack (octaves, lacunarity, gain) and a **domain warp** pass to increase scale richness without obvious stretching.  
5. Your Voronoi-based crack pattern is crisp nearby but shimmers at distance. Which levers—mip bias, **SmoothStep** thresholds, texture resolution, and sampling space—would you adjust, and why?

