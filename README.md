# 3D Game Development — Unreal Materials
## Overview

Core reading material and links for the Semester 7 content.

---

## Recommended Reading

| Title | Purpose | Link |
| :-- | :-- | :-- |
| Unreal Materials — **Naming Convention** | Establish a simple 3‑field naming scheme (**Domain • Surface • Usage**) for **Master Materials** and **Material Instances**, with clear examples and a mind‑map. Includes a table of **material prefixes** (`MM_`, `MI_`, `MF_`, etc.) for consistent Content Browser filtering. | [3dgd_material_naming](/Notes/3dgd_material_naming.md) |
| Unreal Materials — **Noise** | Explain how **Blue**, **Perlin/Gradient**, **Simplex**, **Voronoi/Worley**, and **Brownian/fBm** noise are generated, what they look like, where they’re useful, and their performance trade‑offs. Includes guidance on **pre‑baked textures**, **fBm layering**, **domain warping**, and practical Unreal graph patterns. | [3dgd_noise](/Notes/3dgd_noise.md) |

# Labs

| Lab File | Keywords | Summary | Link |
| :- | :- | :- | :- |
| **Lab 1 – Custom Nodes & Polar UVs** | Custom Node, HLSL, Polar UVs, atan2, UV transforms | Introduces Custom HLSL nodes in Unreal, converting Cartesian UVs to polar coordinates to enable radial distortions, spinning effects, and stylised UV manipulation. | [Lab1_CustomNodesHLSL](Labs/HLSL/Lab1_CustomNodesHLSL.md) |
| **Lab 2 – Animated Raindrops** | Ripples, Loops, Randomisation, Time, Procedural FX | Implements a multi-drop ripple effect using loops, pseudo-random offsets, and time-driven radius expansion. Produces animated raindrop masks suitable for emissive, opacity, or refraction. | [Lab2_RainDrops_HLSL](Labs/HLSL/Lab2_RainDrops_HLSL.md) |
| **Lab 3 – Sobel Edge Detection** | Post-process, Sobel, Edge mask, Convolution | Builds a full-screen Sobel filter in a Custom HLSL node, sampling a 3×3 neighbourhood to compute edge gradients for outlines, toon pipelines, and debugging. | [Lab3_SobelEdgeDetection](Labs/HLSL/Lab3_SobelEdgeDetection.md) |
| ~~Lab 4 – Toon Shading~~ | Toon lighting, Thresholding, ddx/ddy, Halftone | Creates a stylised cel shader using derivative-based edge masks, thresholded lighting bands, soft transitions, and optional halftone overlays for comic rendering. | [Lab4_ToonShading](Labs/HLSL/Lab4_ToonShading.md) |
| **Lab 5 – Heat Distortion** | Post-process, Noise flow, UV warping, Mirage | Uses noise-driven UV offsets to simulate shimmering heat haze. Introduces flow fields, animated distortion, and controlled warping of the post-processing scene. | [Lab5_HeatDistortion_Simple](Labs/HLSL/Lab5_HeatDistortion_Simple.md) |
| **Lab 6 – Pixelation** | Pixel grid, Quantised UVs, Retro FX | Produces a pixelated full-screen effect by quantising screen UVs and resampling scene colour, creating retro handheld and stylised low-resolution looks. | [Lab6_Pixelation](Labs/HLSL/Lab6_Pixelation.md) |
| **Lab 7 – Vignette** | Radial mask, smoothstep, Lens FX | Implements a vignette by computing radial distance from the screen centre and applying soft darkening with smoothstep-based falloff. Artist-controlled radius, softness, and strength. | [Lab7_Vignette](Labs/HLSL/Lab7_Vignette.md) |
| **Lab 8 – Chromatic Aberration** | RGB offsets, Radial direction, Fringe FX | Creates cinematic chromatic aberration by offsetting red and blue channels along radial vectors, with controls for strength, edge intensification, and centre dead-zone. {index=7} | [Lab8_ChromaticAberration](Labs/HLSL/Lab8_ChromaticAberration.md) |
| **Lab 9 – Lens Distortion (Brown–Conrady)** | Brown–Conrady, Radial distortion, k1–k3 | Implements Brown–Conrady radial lens distortion with polynomial UV warping for fisheye or pincushion effects. Includes intensity, zoom, and optical-centre controls. | [Lab9_LensDistortion](Labs/HLSL/Lab9_LensDistortion.md) |
| **Lab 10 – Depth Fog / Atmospheric Haze** | SceneDepth, Fog factor, Exponential falloff | Uses depth sampling to blend scene colour with fog using linear + exponential shaping. Teaches atmospheric effects, readability control, and distance-based blending. | [Lab10_DepthFog](Labs/HLSL/Lab10_DepthFog.md) |
| **Lab 11 – Procedural Patterns (Stripes, Checker, Radial, Waves)** | Procedural UVs, Masks, Waves, Fence patterns | Generates procedural stripes, checkerboards, radial masks, and sine-wave-based fence/weave patterns using Custom HLSL. Highly parameterised for design iteration. | [Lab11_ProceduralPatterns_Waves](Labs/HLSL/Lab11_ProceduralPatterns_Waves.md) |
| **Lab 12 – Procedural Noise & fBm (Stone, Clouds, Marble)** | Noise, fBm, Domain warp, Marble, Clouds | Implements value noise and fBm to generate rock, cloud, and marble materials entirely procedurally, including animation and domain warping. | [Lab12_ProceduralNoise_fBm](Labs/HLSL/Lab12_ProceduralNoise_fBm.md) |
| **Lab 13 – Uniplanar, Biplanar & Triplanar Mapping** | Triplanar mapping, World-space UVs, Projection, Texture blending | Implements uniplanar, biplanar, and triplanar projections in one Custom HLSL node. Uses world position and normals to eliminate stretching on steep slopes, blending planar projections using normal-based weights. Includes parameters for scale, blend sharpness, and mapping mode.  | [Lab13_TriplanarMapping](Labs/HLSL/Lab13_TriplanarMapping.md) |

## Useful Links

* [Computerphile – Finding the Edges (Sobel Operator)](https://www.youtube.com/watch?v=uihBwtPIBxM)

* [EPFL Edge Detector Demo (Sobel, Prewitt, Roberts, etc.](https://bigwww.epfl.ch/demo/ip/demos/edgeDetector/)

* [Wolfram Demonstrations – Sobel Edge Detection Filter](https://demonstrations.wolfram.com/SobelEdgeDetectionFilter/)

