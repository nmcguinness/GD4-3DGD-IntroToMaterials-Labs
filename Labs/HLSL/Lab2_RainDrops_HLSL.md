# Lab 2 – Animated Raindrops with Custom Nodes & HLSL
## Module: 3D Game Development

## Overview

In this lab you will create a procedural **animated raindrop and ripple effect** using a **Custom HLSL node** inside Unreal Engine’s Material Editor.  
This effect draws expanding rings (ripples), randomises their positions, and animates them over time using a pulse-driven radius system.

This lab builds on the Custom Node techniques introduced in **Lab 1 – Introduction to Custom Nodes & HLSL**, extending them with loops, randomisation, and soft fading.

## Learning Outcomes

By completing this lab, you will be able to:

- Implement a multi-input Custom HLSL Node.  
- Use loops (`for`) inside a Custom node.  
- Generate pseudo-random positions for droplets.  
- Animate ripple radius using time-based pulse cycles.  
- Produce raindrop masks for use in opacity, emissive, refraction, or normal distortion.  
- Integrate the mask into a surface material.

## Creating the Raindrop Material Function

### Create a Material Function
1. In the Content Browser: **Right-click → Material → Advanced → Material Function**  
2. Name it:
   ```
   MF_RainDrops
   ```
3. Open the function.

### Add Function Inputs
Add the following **Function Input** nodes:

1. **UV**  
   - Name: `UV`  
   - Type: *Vector 2*

2. **TimeInput**  
   - Name: `TimeInput`  
   - Type: *Scalar*

### Add a Custom Node
1. Add a **Custom** node.  
2. Name it:
   ```
   RainDrops
   ```
3. Set **Output Type = Float 4**.  
4. Add two inputs in the Details panel:
   - `uv` 
   - `time`  
5. Connect:
   - Function Input **UV** → `uv`  
   - Function Input **TimeInput** → `time`

### HLSL Overview
The raindrop effect is produced by:

- Looping many times (e.g., 100 droplets).  
- Generating a random position for each droplet.  
- Using `frac(time / cycle)` to animate a pulse.  
- Growing radius from min to max over the droplet’s life.  
- Creating a ring using distance and smooth fades.  
- Accumulating rings into a single output mask.

### Full HLSL Code
#### Paste this HLSL code into the Custom Node:
```hlsl
float4 result = float4(0, 0, 0, 0);

float radiusMin = 0.05;
float radiusMax = 0.1;
float ringThickness = 1;
float fadeInner = 0.005;
float fadeOuter = 0.001;
float duration = 1.0;

float2 seed = float2(123.456, 789.012);
float2 offsetRange = float2(-1, 1);

float drops = 100;

for (int i = 0; i < drops; i++)
{
    seed = frac(seed * 123.456);

    float2 randOffset = lerp(offsetRange.x, offsetRange.y, seed);

    float cycle = duration + frac(randOffset);
    float pulse = frac(time / cycle);

    float radius = radiusMin + pulse * (radiusMax - radiusMin);

    float2 offset = (uv - 0.5) - randOffset;

    float pointDist = length(offset);

    float radiusLimit = radiusMin + (seed.y) * (radiusMax - radiusMin);

    float alpha = saturate(smoothstep(radius - fadeInner, radius + fadeInner, pointDist));

    alpha *= saturate(1.0 - smoothstep(radiusLimit - fadeOuter, radiusLimit + fadeOuter, pointDist));

    if (pointDist >= radius - ringThickness &&
        pointDist <= radius * ringThickness)
    {
        result += alpha;
    }
}

return saturate(result);
```

## Testing the Function in a Material

### Create a Test Material
1. Create a new Material:
   ```
   M_RainDrops_Test
   ```
2. Add:
   - **TextureCoordinate**  
   - **Time**  
   - **Multiply** (optional rain speed)  
   - **Material Function Call** (MF_RainDrops)

### Connect the Nodes
- TextureCoordinate → `UV`  
- Time (or Time * speed parameter) → `TimeInput`  
- MF_RainDrops output → one of:
  - **Emissive**  
  - **Base Color multiplier**  

Apply the material to a plane or mesh and observe animated ripples.

## Student Tasks

### Adjust Density and Radius
Edit the HLSL:
- `drops`
- `radiusMin`
- `radiusMax`

Observe how droplet count and size affect the rain style.

### Expose Parameters
Convert HLSL constants into **Function Inputs**:
- Drops  
- RadiusMin  
- RadiusMax  
- Duration  

Then connect them to parameters in the calling material, enabling real‑time art control.

### Apply to a Realistic Surface
Use the raindrop mask to:
- Distort normals  
- Drive roughness  

## Reflective Questions

1. Why is `frac(time / cycle)` used to animate droplet radius?  
2. What visual changes occur when increasing or decreasing the `drops` value?  
3. How do smooth fades (`smoothstep`) improve the realism of the ripple?  
4. Why is pseudo-randomisation important in procedural droplet generation?  
5. How might this effect be extended to simulate droplets sliding down glass?

## Useful Links

- [Unreal Engine Material Custom Material Expressions (Official Docs)](https://dev.epicgames.com/documentation/en-us/unreal-engine/custom-material-expressions-in-unreal-engine)

- [Microsoft HLSL Reference](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-pguide)

- [HLSL Tutorial Series (Ben Cloward)](https://www.youtube.com/playlist?list=PL78XDi0TS4lEMvytsE_MoWEpzBcukXv9b)  

## Appendix A — Useful HLSL Functions

### Math Utilities
- **abs(x)** – absolute value  
- **frac(x)** – fractional part  
- **floor(x)** – round down  
- **ceil(x)** – round up  
- **fmod(a, b)** – floating-point modulo  
- **clamp(x, min, max)** – constrain range  
- **saturate(x)** – clamp to [0,1]

### Trigonometry
- **sin(x)**  
- **cos(x)**  
- **tan(x)**

### Vector Operations
- **length(v)** – magnitude  
- **distance(a, b)** – length(a - b)  
- **normalize(v)** – unit vector  
- **dot(a, b)** – projection / angle  
- **cross(a, b)** – perpendicular 3D vector

### Interpolation Functions
- **lerp(a, b, t)** – linear interpolation  
- **smoothstep(a, b, x)** – smooth falloff curve  
- **step(edge, x)** – 0/1 threshold  
- **pow(x, y)** – curve shaping  

### Randomisation Helper
```hlsl
float hash(float2 p)
{
    return frac(sin(dot(p, float2(12.9898, 78.233))) * 43758.5453);
}
```

### 2D Rotation
```hlsl
float2 rotate(float2 uv, float angle)
{
    float s = sin(angle);
    float c = cos(angle);
    return float2(
        uv.x * c - uv.y * s,
        uv.x * s + uv.y * c
    );
}
```

**End of Lab 2 – Animated Raindrops with Custom Nodes & HLSL**
