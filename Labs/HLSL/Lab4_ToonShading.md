# Lab 4 – Toon Shading with Custom Nodes & HLSL  
## Module: 3D Game Development

## Overview

In this lab you will build a **toon-shading material** in Unreal Engine using **Custom HLSL nodes**. This lab introduces three major components of a stylised shader:

1. **Edge/Crevice Masking** using derivatives (`ddx`, `ddy`).  
2. **Threshold-Based Lighting** (Shadow / Midtone / Highlight).  
3. **Optional Halftone Shading** for comic-style rendering.

By completing this lab, you will build a fully controllable toon shader with adjustable tone thresholds, color ramps, softness, edge thickness, edge contrast, and optional halftone overlay.

## Learning Outcomes

You will learn to:

- Use `ddx` / `ddy` to extract curvature and screen-space edge information.  
- Create controllable edge masks via Custom HLSL nodes.  
- Implement toon shading using thresholding and smooth transitions.  
- Blend shadow/mid/highlight colors using Softness control.  
- Combine edge masks and lighting functions into a single stylised output.  
- Optionally incorporate halftone-dot shading for comic rendering.

## Creating the Base Material

### Create a Material
1. Right-click → **Material**  
2. Name it:
   ```
   M_ToonBase
   ```
3. Set **Shading Model → Default Lit** (toon shading overrides lighting manually).  
4. Ensure **Use Material Attributes** is *off* for simplicity.

## Creating the Edge/Curvature Mask

This section demonstrates calculating edge details by measuring change in the surface normals using derivative instructions.  

### Add a Custom Node for Edge Mask
1. Add a **Custom Node** called:
   ```
   EdgeMask
   ```
2. Add inputs:
   - `NormalsWS` (Vector3) → connect from **PixelNormalWS**  
   - `Thickness` (Scalar)  
   - `Contrast` (Scalar)

3. Paste this HLSL code (from page 7 and the extracted file) fileciteturn3file1:

```hlsl
float edgeDetails = length(ddx(NormalsWS)) + 
                    length(ddy(NormalsWS));

edgeDetails /= (Thickness + 1.0f);  
edgeDetails  = (edgeDetails - 0.5f) * Contrast + 0.5f;

return saturate(edgeDetails);
```

### Parameter Setup
- Convert `Thickness` and `Contrast` to **Parameters**.  
- Add each of the paramaters to a group:
  - Group: **Edges**
  - Parameters: Thickness, Contrast

### Preview
Connect `EdgeMask` output to **Emissive Color** temporarily to verify edges are appearing.

## Creating the Toon Lighting Function

We now build a thresholded lighting model to classify each pixel into:
- Shadow  
- Midtone  
- Highlight  

### Add a Toon Lighting Custom Node
1. Add a Custom Node named:
   ```
   ToonLight
   ```
2. Inputs:
   - `NormalsWS`  
   - `LightDir`  
   - `Shadow` (Scalar)  
   - `Mid` (Scalar)  
   - `High` (Scalar)  
   - `In1` (Vector3) – Shadow Color  
   - `In2` (Vector3) – Midtone Color  
   - `In3` (Vector3) – Highlight Color  
   - `Softness` (Scalar)

3. Paste HLSL from the provided file M_SimpleToon_CustomLight.txt fileciteturn3file4:

```hlsl
float lightIntensity = max(dot(NormalsWS, LightDir), 0);

float3 color;

if (lightIntensity > High) 
    color = In3;  
else if (lightIntensity > Mid) 
    color = lerp(In2, In3, smoothstep(0, Softness, (lightIntensity - Mid) / (High - Mid)));
else if (lightIntensity > Shadow)
    color = lerp(In1, In2, smoothstep(0, Softness, (lightIntensity - Shadow) / (Mid - Shadow)));
else 
    color = In1;

return color;
```

### Grouping Parameters
Use two parameter groups:  
- **Gradient** – In1, In2, In3  
- **Size and Falloff** – Shadow, Mid, High, Softness

### Preview
Connect the ToonLight output directly to **Base Color** to preview.

## Combining Edge Mask with Lighting

We will next combine the lighting result with the inverted edge mask.

### Node Logic
1. **1 – EdgeMask**  
2. Invert edge mask:  
   ```
   1 - EdgeMask
   ```
3. Multiply by toon lighting output.  
4. Optionally saturate.

### Build Combined Output
```
FinalColor = saturate( ToonLight * (1 - EdgeMask) );
```

Set **Base Color** = FinalColor.

## Optional: Halftone Pattern Overlay

### Add Halftone Custom Node
Inputs:
- `color` (Vector3)
- `uv` (Vector2)
- `tiling` (Scalar)
- `maxSize` (Scalar)

### Paste halftone code

```hlsl
float luminance = dot(color, float3(0.299, 0.587, 0.114));

float dist = length(frac(uv * tiling) - 0.5);
float radius = (1.0 - luminance) * maxSize;

return lerp(1, 0.5, step(dist, radius));
```

- Multiply this output by the toon-lighting color.
- Use screen-space UVs or texture UVs depending on intended look.

## Final Material Output

Set:
- **Base Color** = combined toon lighting *(with optional halftone overlay)*  
- **Roughness** = 0.5 (or artistic choice)  
- **Specular** = 0 (for toon shading)

You now have a fully functioning, stylised toon shader.

## Student Tasks

### Adjust Tone Thresholds
Explore how Shadow, Mid, High alter banding.

### Experiment with Halftone Patterns
Try screen-space vs UV-space mapping for dots.

## Reflective Questions

1. Why does toon shading benefit from thresholded lighting instead of continuous shading?  
2. What advantages do `ddx` and `ddy` offer for stylised rendering?  
3. How does adjusting Softness affect the shading style?  
4. What visual differences occur when blending lighting and edge masks?  
5. How can halftone shading enhance stylised material aesthetics?

## Useful Links

- [Unreal Engine Advanced Cel Shader](https://dev.epicgames.com/community/learning/tutorials/qZl2/unreal-engine-advanced-cel-shader)
  
- [Unreal Engine Material Custom Material Expressions (Official Docs)](https://dev.epicgames.com/documentation/en-us/unreal-engine/custom-material-expressions-in-unreal-engine)

- [Microsoft HLSL Reference](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-pguide)

- [HLSL Tutorial Series (Ben Cloward)](https://www.youtube.com/playlist?list=PL78XDi0TS4lEMvytsE_MoWEpzBcukXv9b)

## Appendix A — Background: Technical Notes on Toon Shading Filters

### 1. Derivative-Based Edge Detection (`ddx`, `ddy`)
- Measures change in normals across screen space.
- High variation gives astrong edge.
- Very cheap vs full Sobel filter.
- Ideal for stylised outlines that match geometry curvature.

### 2. Thresholded Lighting Models
- Creates clear “bands” of tone.
- Can emulate anime, comic, and cel shading styles.
- Softness blends edges for smoother gradients.

### 3. Halftone Dot Functions
- Common in printed comics.
- Dot radius scales with luminance.
- Provides retro, stylised look.
- Works in screen UV or texture UV space.
