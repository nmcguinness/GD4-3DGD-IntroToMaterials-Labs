# Lab 5 – Heat Distortion / Mirage (Node-Based Noise + Simple HLSL)
## Module: 3D Game Development

## Overview
In this lab you will implement a heat-distortion / mirage effect as a Post-Process Material in Unreal Engine.  
This simplified version uses **Material graph noise** to generate the distortion flow and keeps the **Custom HLSL node minimal**, making it ideal for students new to HLSL.

The effect simulates shimmering hot air rising from surfaces by warping the final rendered frame.



## Learning Outcomes
- Use a Post-Process Material in Unreal Engine.  
- Pass values from Material nodes into a Custom HLSL node.  
- Distort screen UVs in HLSL and re-sample the scene.  
- Control distortion using parameters (strength, scale, speed).  



## 1. Create a Post Process Volume
1. Drag a Post Process Volume into your level.  
2. Enable **Infinite Extent (Unbound)** → *true*.  
3. Add a **Post Process Material** slot (Asset Reference).  



## 2. Create the Material
1. Create a Material named **M_HeatDistortion_Simple**.  
2. Set **Material Domain → Post Process**.  
3. Output will go to **Emissive Color**.



## 3. Create a Screen-UV Helper Custom Node
Add a Custom node:

- **Name:** `GetScreenUVs`  
- **Output Type:** Float2  
- **Code:**  
```hlsl
return GetSceneTextureUV(Parameters);
```

This gives correct 0–1 UV coordinates for the current pixel.



## 4. Build Node-Based Noise for Distortion Flow

### 4.1 Add Parameters
- `DistortionStrength` = 0.008  
- `DistortionScale` = 10.0  
- `DistortionSpeed` = 1.0  

### 4.2 Scale Screen UVs
Multiply `GetScreenUVs` by `DistortionScale` → `ScaledUV`.

### 4.3 Animate the Noise
1. Add a **Time** node.  
2. Multiply Time by `DistortionSpeed`.  
3. Create two offsets:  
   - `AnimX = Time * 0.5`  
   - `AnimY = Time * 0.3`  
4. Append them → `TimeOffset`.  
5. Add `ScaledUV + TimeOffset` → `AnimatedNoiseUV`.

### 4.4 Create Two Noise Channels
Use two **Noise** nodes:

- `NoiseX = Noise(AnimatedNoiseUV)`  
- `NoiseY = Noise(AnimatedNoiseUV + (13.37, 42.0))`  

Subtract 0.5 from each to remap them to **[-0.5, +0.5]**:

- `CenteredNoiseX = NoiseX - 0.5`  
- `CenteredNoiseY = NoiseY - 0.5`  

Append → `Flow (float2)`.

This `Flow` input will be fed into the Custom HLSL node.



## 5. Main Custom HLSL Node (Simple Distortion)

### Inputs
| Name | Type | Connect |
|:-|:-|:-|
| `uv` | Float2 | Output of `GetScreenUVs` |
| `flow` | Float2 | Append node (`CenteredNoiseX`,`CenteredNoiseY`) |
| `distortionStrength` | Float1 | Parameter |

### HLSL Code 
```hlsl
// Inputs:
//   uv                 : float2 screen-space UV (0..1)
//   flow               : float2 distortion vector from the material graph
//   distortionStrength : scalar intensity of the distortion

// Start from the original screen UV
float2 baseUV = uv;

// Apply the flow vector, scaled by the distortion strength parameter
float2 offsetUV = baseUV + flow * distortionStrength;

// Clamp the UVs to the valid screen range [0, 1]
// This prevents sampling outside the render target
offsetUV = clamp(offsetUV, 0.0, 1.0);

// Sample the rendered scene at the distorted coordinates
// 14 = PostProcessInput0 (scene color) in Unreal's post-process materials
float3 distortedColor = SceneTextureLookup(offsetUV, 14, false).rgb;

// Return the distorted scene color for this pixel
return distortedColor;
```

### Output
Connect the Custom node output → **Emissive Color**.



## 6. Assign the Material
Select the Post Process Volume → Post Process Materials → assign **M_HeatDistortion_Simple**.  
Set **Blend Weight = 1.0**.



## 7. Student Tasks

### Task 1 – Create Desert and Underwater Variants
Experiment with:
- Strength: 0.003–0.02  
- Scale: 5–20  
- Speed: 0.2–3.0  

### Task 2 – Height Mask (Optional)
Use `UV.y` from `GetScreenUVs` to fade distortion near the bottom of the screen.

### Task 3 – Combine Effects
Try:
- Toon shading + heat shimmer  
- Edge outlines + shimmer  



## 8. Reflective Questions
1. Why is the distortion noise generated in the Material graph instead of HLSL?  
2. How does DistortionScale affect wave size?  
3. What happens if offsetUV is not clamped?  
4. Why is a 2D flow vector better than a single noise value?  
5. Where would this effect enhance gameplay, and where might it cause visual discomfort?


