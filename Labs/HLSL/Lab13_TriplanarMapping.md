# Lab 13 – Uniplanar, Biplanar and Triplanar Mapping with Custom Nodes & HLSL
## Module: 3D Game Development

## Overview

In this lab you will implement **uniplanar, biplanar and triplanar texture mapping** using a single **Custom HLSL node** in a **surface material**.

Traditional UV mapping often stretches textures badly on steep slopes or on meshes with poor UV layouts (for example, cliffs, rocks, and modular environment pieces). Triplanar mapping avoids this by **projecting the texture from multiple directions** (X, Y, Z axes) and blending them based on the surface normal.

You will:

- project a texture in world space (uniplanar),
- blend between two projections (biplanar),
- generalise this to three projections weighted by the world normal (triplanar).

The same Custom node will contain all three methods, controlled by a parameter.

## Learning Outcomes

By completing this lab, you will be able to:

- Explain why triplanar mapping is used in games to reduce texture stretching.
- Use world-space position and normal in a Custom HLSL node.
- Implement uniplanar, biplanar and triplanar projections in one HLSL function.
- Blend projected textures based on the absolute world normal.
- Expose parameters for texture scale, blend sharpness and mapping mode.

## 1. Theory – Why Triplanar Mapping?

### UV stretching on steep slopes

Standard UVs are fixed to the mesh. If a surface is very steep or vertical, and the UVs were laid out flat, then:

- textures get **squashed** when seen from the side,
- or **stretched** when the surface is almost vertical.

This is very noticeable on:

- terrain with cliffs,
- rocks and boulders,
- modular environment pieces with reused textures.

### Projecting textures instead of relying on UVs

Instead of trusting the mesh UVs, we can use **world-space coordinates** as our “UVs”.

- For a **top-down projection**, we might use world X and Y as our UV axes.
- For a **side projection**, we might use X/Z or Y/Z.

This is called **planar projection**.

### From uniplanar to biplanar to triplanar

- **Uniplanar mapping**  
  Project texture from a single direction (for example, top-down using world X/Y).

- **Biplanar mapping**  
  Blend between two planar projections (for example, top-down and one side axis).

- **Triplanar mapping**  
  Blend between **three** planar projections (X, Y, Z) based on how much the surface normal points in each direction.

Key idea:

- Take the **absolute value of the normal** (so front/back faces behave the same).
- Use the **X, Y, Z components** as blend weights for the three projections.
- Normalise the weights so they sum to 1.

This produces a seamless blend across edges, with minimal stretching.

## 2. Material Setup

Create a new material:

```text
M_TriplanarMapping
```

Set:

- **Material Domain → Surface**
- **Shading Model → Default Lit** (so we can still use lighting)

You will output the triplanar colour into **Base Color**.

### Parameters

Create these parameters:

| Parameter Name | Type | Default | Description |
| :- | :- | :- | :- |
| `BaseTexture` | Texture2D | (choose a tiling rock/ground texture) | Texture to project |
| `TextureScale` | Scalar | 0.1 | World-space tiling scale |
| `BlendSharpness` | Scalar | 2.0 | Controls how sharp the triplanar blend is |
| `MappingMode` | Scalar | 0.0 | 0 = uniplanar, 1 = biplanar, 2 = triplanar |

Notes:

- `TextureScale` will be multiplied with world position.
- `MappingMode` will be used inside the HLSL to switch behaviour.

### World-space inputs

Add:

- **AbsoluteWorldPosition** node → gives world position `float3`.
- **PixelNormalWS** node → gives world-space normal `float3`.

You will connect:

- AbsoluteWorldPosition → Custom node `worldPos`
- PixelNormalWS → Custom node `worldNormal`
- BaseTexture → Custom node `baseTex`
- BaseTexture’s sampler → Custom node `baseTexSampler`

## 3. Custom Node Setup

Add a **Custom** node named `PlanarMapping`.

Set:

- **Output Type → Float3** (we output a colour)

Add these **Inputs**:

| Input Name | Type | Connect From |
| :- | :- | :- |
| `worldPos` | Float3 | AbsoluteWorldPosition |
| `worldNormal` | Float3 | PixelNormalWS |
| `baseTex` | Texture2D | `BaseTexture` parameter |
| `baseTexSampler` | SamplerState | `BaseTexture` sampler |
| `textureScale` | Float1 | `TextureScale` |
| `blendSharpness` | Float1 | `BlendSharpness` |
| `mappingMode` | Float1 | `MappingMode` |

Make sure the `baseTex` input is set to **TextureObject** and `baseTexSampler` is of type **SamplerState** in the Custom node settings.

## 4. HLSL Code – Uniplanar, Biplanar, Triplanar 

Paste the following fully commented HLSL into the `PlanarMapping` Custom node.

```hlsl
// Inputs:
//   worldPos      : float3 world-space position of the pixel
//   worldNormal   : float3 world-space normal of the pixel
//   baseTex       : Texture2D object for the base texture
//   baseTexSampler: sampler for baseTex
//   textureScale  : scalar tiling factor applied to world-space coordinates
//   blendSharpness: controls how sharp triplanar blending is (>=1)
//   mappingMode   : 0=uniplanar, 1=biplanar, 2=triplanar

// Helper function to sample the texture given a planar UV.
// We use the Unreal-specific Texture2DSample macro.
float3 SampleBaseTexture(float2 uv)
{
    // Sample the base texture at the given UV and return RGB.
    return Texture2DSample(baseTex, baseTexSampler, uv).rgb;
}

float3 main()
{
    // Step 1: Prepare basic data.

    // Normalise the world normal so its length is 1.
    float3 n = normalize(worldNormal);

    // Take the absolute value so back-facing directions behave like front-facing.
    // This avoids hard seams when the normal flips sign.
    float3 an = abs(n);

    // Scale world position by the textureScale parameter.
    // Smaller textureScale gives more tiling; larger gives less.
    float3 p = worldPos * textureScale;

    // Step 2: Build planar UVs for each axis.
    // We treat Unreal as Z-up, so:
    // - X/Y plane: "top" projection
    // - Y/Z plane: side projection in X direction
    // - X/Z plane: side projection in Y direction

    // UV when projecting along +Z/-Z (top-down): use X and Y as UV axes.
    float2 uvZ = float2(p.x, p.y);

    // UV when projecting along +X/-X: use Y and Z as UV axes.
    float2 uvX = float2(p.y, p.z);

    // UV when projecting along +Y/-Y: use X and Z as UV axes.
    float2 uvY = float2(p.x, p.z);

    // Step 3: Sample the texture for each projection.
    float3 texZ = SampleBaseTexture(uvZ); // top-down projection
    float3 texX = SampleBaseTexture(uvX); // X-facing projection
    float3 texY = SampleBaseTexture(uvY); // Y-facing projection

    // We will build three different mapping methods and then choose
    // one based on mappingMode.

    // -----------------------------
    // Uniplanar mapping (mode 0)
    // -----------------------------
    // Use a single planar projection (top-down along Z).
    float3 colorUniplanar = texZ;

    // -----------------------------
    // Biplanar mapping (mode 1)
    // -----------------------------
    // Blend between top-down (Z) and a single side projection (X).
    // We use the normal's Z and X components to decide weights.

    // Raw weights from absolute normal components for Z and X.
    float wZ_bi = an.z;
    float wX_bi = an.x;

    // Normalise the two weights so they sum to 1 (avoid divide by zero).
    float sum_bi = max(wZ_bi + wX_bi, 1e-4);
    wZ_bi /= sum_bi;
    wX_bi /= sum_bi;

    // Blend the two projections using the biplanar weights.
    float3 colorBiplanar = texZ * wZ_bi + texX * wX_bi;

    // -----------------------------
    // Triplanar mapping (mode 2)
    // -----------------------------
    // Blend between all three projections (X, Y, Z) using their
    // absolute normal components as weights.

    // Start with absolute normal as weights.
    float3 weights = an;

    // Optionally sharpen the blend by raising weights to a power.
    // blendSharpness = 1.0  → soft blend
    // blendSharpness > 1.0  → sharper, more axis-aligned
    weights = pow(weights, blendSharpness);

    // Normalise weights so x + y + z = 1.
    float weightSum = max(weights.x + weights.y + weights.z, 1e-4);
    weights /= weightSum;

    // Blend the three projections using the final weights.
    float3 colorTriplanar =
        texX * weights.x +
        texY * weights.y +
        texZ * weights.z;

    // -----------------------------
    // Select output based on mappingMode
    // -----------------------------
    // mappingMode:
    //   0 → uniplanar
    //   1 → biplanar
    //   2 → triplanar (default for production use)

    float3 finalColor = colorTriplanar; // default

    if (mappingMode < 0.5)
    {
        finalColor = colorUniplanar;
    }
    else if (mappingMode < 1.5)
    {
        finalColor = colorBiplanar;
    }
    else
    {
        finalColor = colorTriplanar;
    }

    // Return the chosen mapping colour.
    return finalColor;
}
```

## 5. Connect the Custom Node to the Material

Connect:

- `PlanarMapping` Custom node output → **Base Color**

Leave Metallic, Roughness, Normal, etc., at suitable defaults or plug constants as needed.

Apply the material to:

- a large cube,
- a sphere,
- a landscape or irregular mesh.

Observe the difference between **MappingMode** 0, 1 and 2.

## 6. Material Instances and Experiments

Create a Material Instance:

```text
MI_TriplanarMapping
```

Test:

- `MappingMode = 0` (uniplanar)  
  - Notice strong stretching on vertical sides.

- `MappingMode = 1` (biplanar)  
  - Some blending between top and one side; reduces stretching on certain angles.

- `MappingMode = 2` (triplanar)  
  - Texture appears much more uniform across all angles.

Adjust:

- `TextureScale` to control tiling in world units.
- `BlendSharpness` from 1.0 to 8.0 and see how edges become sharper or softer.

## 7. Student Tasks

### Task 1 – Compare UV vs Triplanar

Apply:

- A standard UV-mapped material using the same texture.
- `MI_TriplanarMapping` with `MappingMode = 2`.

Compare:

- cliffs or steep surfaces,
- stretched vs consistent texel density.

Make notes on when triplanar is visually superior.

### Task 2 – Add a Second Texture for Vertical Surfaces

Extend the HLSL to support:

- one texture for “top” surfaces (using Z projection),
- a second texture for vertical surfaces (X and Y projections).

Use the weights in the triplanar block to blend rock vs grass or snow vs rock.

### Task 3 – Triplanar Normal Mapping (Design Exercise)

Outline how you would:

- feed a Normal Map texture into the same Custom node,
- sample normal maps along X, Y, Z,
- reconstruct and blend them using the same weights,
- transform them into world-space or tangent-space as needed.

You do not need to fully implement this in code, but describe the steps clearly.

## 8. Reflective Questions

1. Why does triplanar mapping significantly reduce texture stretching compared to simple UV mapping on steep slopes?
2. What role do the absolute normal components play in deciding how much each projection contributes?
3. How does increasing `BlendSharpness` change the look of the triplanar blend? When might softer vs sharper blends be preferable?
4. In what situations would uniplanar or biplanar mapping still be sufficient, instead of full triplanar?
5. How could triplanar mapping be combined with procedural noise (from Lab 12) to create more varied and organic surface materials?

