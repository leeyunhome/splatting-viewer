# Spark.js Gaussian Splatting Rendering Pipeline

## Overview
This document explains how Spark.js renders 3D Gaussian Splats with direct links to the source code.

**Repository**: https://github.com/sparkjsdev/spark

---

## 1. Data Structure - Packed Splats

### PackedSplats Class
**File**: [`src/PackedSplats.ts`](https://github.com/sparkjsdev/spark/blob/main/src/PackedSplats.ts)

Each Gaussian splat is packed into **16 bytes** (uvec4 = 4 × uint32):
- **center xyz**: float16 (3 × 2 bytes) - Lines 72-90
- **scales xyz**: quantized log values
- **quaternion**: quantized rotation (4 values)  
- **rgba**: color + opacity

Stored in a `DataArrayTexture` (2048×2048×depth) for efficient GPU access.

**Key Functions**:
- [`PackedSplats.getTexture()`](https://github.com/sparkjsdev/spark/blob/main/src/PackedSplats.ts#L461-L555) - Returns the texture
- [`PackedSplats.generate()`](https://github.com/sparkjsdev/spark/blob/main/src/PackedSplats.ts#L635-L652) - Generates splats on GPU

---

## 2. GLSL Shader Definitions

### Packing/Unpacking Functions
**File**: [`src/shaders/splatDefines.glsl`](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatDefines.glsl)

**Key Functions**:
- [`packSplat()`](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatDefines.glsl#L227-L238) - Lines 227-238
  - Packs center, scales, quaternion, rgba into uvec4
  
- [`unpackSplatEncoding()`](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatDefines.glsl#L227-L238) - Lines 238+
  - Unpacks uvec4 back into individual components

---

## 3. Vertex Shader - Core Rendering Logic

### Main Vertex Shader
**File**: [`src/shaders/splatVertex.glsl`](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatVertex.glsl)

This is where the **magic happens**! Each Gaussian is transformed from 3D to 2D.

### Step 1: Fetch and Unpack Splat Data
**Lines**: [41-70](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatVertex.glsl#L41-L70)

```glsl
// Get splat index from instance ID
ivec3 texCoord = ivec3(
    splatIndex & SPLAT_TEX_WIDTH_MASK,
    (splatIndex >> SPLAT_TEX_WIDTH_BITS) & SPLAT_TEX_HEIGHT_MASK,
    splatIndex >> SPLAT_TEX_LAYER_BITS
);

// Fetch packed data from texture
uvec4 packed = texelFetch(packedSplats, texCoord, 0);
unpackSplat(packed, center, scales, quaternion, rgba);
```

### Step 2: Transform to View Space
**Lines**: [100-122](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatVertex.glsl#L100-L122)

```glsl
// Transform center to view space
vec3 viewCenter = quatVec(renderToViewQuat, center) + renderToViewPos;
vec4 clipCenter = projectionMatrix * vec4(viewCenter, 1.0);

// Frustum culling
float clip = clipXY * clipCenter.w;
if (abs(clipCenter.x) > clip || abs(clipCenter.y) > clip) {
    return; // Discard splat outside view
}

// Transform quaternion to view space
vec4 viewQuaternion = quatQuat(renderToViewQuat, quaternion);
```

### Step 3: Build 3D Covariance Matrix
**Lines**: [122-145](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatVertex.glsl#L122-L145)

```glsl
// Compute the 3D covariance matrix of the splat
mat3 RS = scaleQuaternionToMatrix(scales, viewQuaternion);
mat3 cov3D = RS * transpose(RS);
```

This creates the 3D ellipsoid representation of the Gaussian.

### Step 4: Project to 2D (KEY ALGORITHM!)
**Lines**: [145-167](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatVertex.glsl#L122-L145)

```glsl
// Compute focal length in pixels
vec2 focal = 0.5 * renderSize * vec2(projectionMatrix[0][0], projectionMatrix[1][1]);

// Compute Jacobian of projection at splat center
mat3 J = computeJacobianMatrix(viewCenter, focal);

// PROJECT 3D COVARIANCE TO 2D SCREEN SPACE!
mat3 cov2D = transpose(J) * cov3D * J;
float a = cov2D[0][0];
float d = cov2D[1][1];
float b = cov2D[0][1];
```

**This is the core innovation**: The Jacobian projects the 3D Gaussian ellipsoid onto the 2D screen plane.

### Step 5: Anti-Aliasing and Blur
**Lines**: [167-186](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatVertex.glsl#L167-L186)

```glsl
// Add pre-blur (optional)
a += preBlurAmount;
d += preBlurAmount;

// Add blur for anti-aliasing (typically 0.3)
float detOrig = a * d - b * b;
a += fullBlurAmount;
d += fullBlurAmount;
float det = a * d - b * b;

// Adjust opacity to conserve energy
float blurAdjust = sqrt(max(0.0, detOrig / det));
rgba.a *= blurAdjust;

if (rgba.a < minAlpha) {
    return; // Discard transparent splats
}
```

### Step 6: Compute 2D Ellipse Parameters
**Lines**: [187-207](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatVertex.glsl#L187-L207)

```glsl
// Compute eigenvalues (ellipse axes lengths)
float eigenAvg = 0.5 * (a + d);
float eigenDelta = sqrt(max(0.0, eigenAvg * eigenAvg - det));
float eigen1 = eigenAvg + eigenDelta;
float eigen2 = eigenAvg - eigenDelta;

// Compute eigenvector (ellipse orientation)
vec2 eigenVec1 = normalize(vec2((abs(b) < 0.001) ? 1.0 : b, eigen1 - a));

// Compute radii in pixels (bounded by min/max pixel radius)
vec2 radii = clamp(
    maxStdDev * sqrt(max(vec2(0.001), vec2(eigen1, eigen2))),
    vec2(minPixelRadius),
    vec2(maxPixelRadius)
);
```

### Step 7: Generate Quad Vertices
**Lines**: [207+](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatVertex.glsl#L187-L207)

```glsl
// Create rotation matrix for ellipse orientation
mat2 ellipseRotation = mat2(
    eigenVec1.x, -eigenVec1.y,
    eigenVec1.y, eigenVec1.x
);

// Generate quad vertex at corner of ellipse
vec2 ndcOffset = (ellipseRotation * (position.xy * radii)) / (0.5 * renderSize);
vec3 ndcPos = ndcCenter + vec3(ndcOffset, 0.0);

// Convert back to clip space
gl_Position = vec4(ndcPos * clipCenter.w, clipCenter.w);
vSplatUv = position.xy * maxStdDev;
vRgba = rgba;
```

Each splat is rendered as a **quad (2 triangles)** oriented along the ellipse axes.

---

## 4. Fragment Shader - Per-Pixel Evaluation

### Main Fragment Shader
**File**: [`src/shaders/splatFragment.glsl`](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatFragment.glsl)

**Lines**: [1-42](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatFragment.glsl#L1-L42) - Setup and uniforms

**Lines**: [42-98](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatFragment.glsl#L56-L79) - Main evaluation

### Gaussian Falloff
```glsl
void main() {
    vec4 rgba = vRgba;
    
    // Compute squared distance from ellipse center (in normalized coordinates)
    float z = dot(vSplatUv, vSplatUv);
    
    // Discard pixels outside Gaussian bounds
    if (z > (maxStdDev * maxStdDev)) {
        discard;
    }
    
    // Apply Gaussian falloff: exp(-0.5 * z²)
    rgba.a *= mix(1.0, exp(-0.5 * z), falloff);
    
    if (rgba.a < minAlpha) {
        discard;
    }
    
    // Output color
    fragColor = rgba;
}
```

### Stochastic Rendering (Optional)
**Lines**: [79-98](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatFragment.glsl#L79-L98)

```glsl
if (stochastic) {
    // Generate pseudo-random number based on pixel position
    uint hash = computeHash(coord, vSplatIndex, time);
    float rand = float(hash) / 4294967296.0;
    
    // Stochastic alpha testing
    if (rand < rgba.a) {
        fragColor = vec4(rgba.rgb, 1.0);
    } else {
        discard;
    }
}
```

---

## 5. Spherical Harmonics (View-Dependent Lighting)

### SH Evaluation Functions
**File**: [`src/SplatMesh.ts`](https://github.com/sparkjsdev/spark/blob/main/src/SplatMesh.ts)

**SH1 (1st order)**: [Lines 762-781](https://github.com/sparkjsdev/spark/blob/main/src/SplatMesh.ts#L762-L781)
```glsl
vec3 evaluateSH1(Gsplat gsplat, usampler2DArray sh1, vec3 viewDir) {
    // Extract packed SH coefficients
    uvec2 packed = texelFetch(sh1, splatTexCoord(gsplat.index), 0).rg;
    vec3 sh1_0 = unpackSH(packed.x);
    vec3 sh1_1 = unpackSH(...);
    vec3 sh1_2 = unpackSH(...);
    
    // Evaluate SH basis functions
    return sh1_0 * (-0.4886025 * viewDir.y)
         + sh1_1 * (0.4886025 * viewDir.z)
         + sh1_2 * (-0.4886025 * viewDir.x);
}
```

**SH2 (2nd order)**: [Lines 895-917](https://github.com/sparkjsdev/spark/blob/main/src/SplatMesh.ts#L897-L917)

**SH3 (3rd order)**: [Lines 813-829](https://github.com/sparkjsdev/spark/blob/main/src/SplatMesh.ts#L813-L829)

### Integration in Rendering Pipeline
**Lines**: [372-435](https://github.com/sparkjsdev/spark/blob/main/src/SplatMesh.ts#L372-L435)

```typescript
constructGenerator(context: SplatMeshContext) {
    let gsplat = readPackedSplat(this.packedSplats.dyno, index);
    
    if (this.maxSh >= 1) {
        // Calculate view direction in object space
        const viewDir = normalize(sub(center, viewCenterInObject));
        
        // Evaluate SH and add to RGB
        let rgb = evaluateSH1(gsplat, sh1Texture, viewDir);
        if (this.maxSh >= 2) {
            rgb = add(rgb, evaluateSH2(gsplat, sh2Texture, viewDir));
        }
        if (this.maxSh >= 3) {
            rgb = add(rgb, evaluateSH3(gsplat, sh3Texture, viewDir));
        }
        
        // Add SH lighting to base color
        rgba = add(rgba, extendVec(rgb, 0.0));
    }
}
```

---

## 6. Sorting and Rendering

### SparkRenderer Class
**File**: [`src/SparkRenderer.ts`](https://github.com/sparkjsdev/spark/blob/main/src/SparkRenderer.ts)

### Initialization
**Lines**: [254-279](https://github.com/sparkjsdev/spark/blob/main/src/SparkRenderer.ts#L254-L279)

```typescript
constructor(options: SparkRendererOptions) {
    const material = new THREE.ShaderMaterial({
        glslVersion: THREE.GLSL3,
        vertexShader: shaders.splatVertex,
        fragmentShader: shaders.splatFragment,
        premultipliedAlpha: true,
        transparent: true,
        depthTest: true,
        depthWrite: false,  // No depth write for alpha blending
    });
}
```

### Update and Render
**Lines**: [548-566](https://github.com/sparkjsdev/spark/blob/main/src/SparkRenderer.ts#L548-L566)

```typescript
onBeforeRender(renderer, scene, camera) {
    // Update uniforms
    this.uniforms.renderSize.value.set(width, height);
    this.uniforms.renderToViewQuat.value = quaternion;
    this.uniforms.renderToViewPos.value = position;
    
    // Set stochastic vs transparent rendering mode
    this.material.transparent = !viewpoint.stochastic;
    this.material.depthWrite = viewpoint.stochastic;
}
```

### Geometry Setup
**File**: [`src/SplatGeometry.ts`](https://github.com/sparkjsdev/spark/blob/main/src/SplatGeometry.ts)

**Lines**: [0-44](https://github.com/sparkjsdev/spark/blob/main/src/SplatGeometry.ts#L0-L44)

```typescript
// Each splat instance = 2 triangles (quad)
const QUAD_VERTICES = new Float32Array([
    -1, -1, 0,  1, -1, 0,  1, 1, 0,  -1, 1, 0
]);
const QUAD_INDICES = new Uint16Array([0, 1, 2, 0, 2, 3]);

// Instance attribute contains splat ordering for depth sorting
class SplatGeometry extends THREE.InstancedBufferGeometry {
    // ordering buffer updated each frame based on camera position
}
```

---

## 7. Dynamic Splat Modification (Dyno System)

### Dyno Block System
**File**: [`src/dyno/splats.ts`](https://github.com/sparkjsdev/spark/blob/main/src/dyno/splats.ts)

**Gsplat Structure**: [Lines 333-338](https://github.com/sparkjsdev/spark/blob/main/src/dyno/splats.ts#L333-L338)

```glsl
struct Gsplat {
    uint flags;      // Active/inactive state
    int index;       // Splat index
    vec3 center;     // Position
    vec3 scales;     // Size (3 axes)
    vec4 quaternion; // Rotation
    vec4 rgba;       // Color + opacity
};
```

**Split/Combine Functions**: [Lines 429-475](https://github.com/sparkjsdev/spark/blob/main/src/dyno/splats.ts#L429-L475)

```typescript
// Split a Gsplat into individual components
splitGsplat(gsplat) => { center, scales, quaternion, rgba, ... }

// Combine components back into a Gsplat
combineGsplat({ center, scales, quaternion, rgba }) => gsplat
```

### Example: Real-time Modification
**File**: [`examples/splat-shader-effects/index.html`](https://github.com/sparkjsdev/spark/blob/main/examples/splat-shader-effects/index.html)

**Lines**: [56-82](https://github.com/sparkjsdev/spark/blob/main/examples/splat-shader-effects/index.html#L56-L82)

```javascript
cat.objectModifier = dyno.dynoBlock(
    { gsplat: dyno.Gsplat },
    { gsplat: dyno.Gsplat },
    ({ gsplat }) => {
        const d = new dyno.Dyno({
            inTypes: { gsplat: dyno.Gsplat, t: "float" },
            outTypes: { gsplat: dyno.Gsplat },
            globals: () => [`
                vec3 hash(vec3 p) { /* custom GLSL */ }
            `],
            statements: ({ inputs, outputs }) => [`
                // Modify splat in GLSL
                ${outputs.gsplat}.center += sin(${inputs.t}) * 0.1;
                ${outputs.gsplat}.rgba.a *= customEffect();
            `]
        });
        return { gsplat: d.apply({ gsplat, t }).gsplat };
    }
);
```

---

## 8. PLY File Loading

### PLY Reader
**File**: [`src/ply.ts`](https://github.com/sparkjsdev/spark/blob/main/src/ply.ts)

**Lines**: [221-282](https://github.com/sparkjsdev/spark/blob/main/src/ply.ts#L221-L282)

```typescript
class PlyReader {
    parseSplats(splatCallback, shCallback) {
        // Parse PLY header and data
        // Extract: position, scale, rotation, color, SH coefficients
        // Call callback for each splat
    }
}
```

### Splat Data Structure
**File**: [`src/SplatLoader.ts`](https://github.com/sparkjsdev/spark/blob/main/src/SplatLoader.ts)

**Lines**: [575-665](https://github.com/sparkjsdev/spark/blob/main/src/SplatLoader.ts#L575-L665)

```typescript
export class SplatData {
    numSplats: number;
    centers: Float32Array;      // xyz positions
    scales: Float32Array;       // xyz scales
    quaternions: Float32Array;  // wxyz rotations
    opacities: Float32Array;    // alpha values
    colors: Float32Array;       // rgb colors
    sh1?: Float32Array;         // 1st order SH
    sh2?: Float32Array;         // 2nd order SH
    sh3?: Float32Array;         // 3rd order SH
}
```

---

## 9. Uniforms Reference

### Complete Uniforms List
**File**: [`src/SparkRenderer.ts`](https://github.com/sparkjsdev/spark/blob/main/src/SparkRenderer.ts)

**Lines**: [339-400](https://github.com/sparkjsdev/spark/blob/main/src/SparkRenderer.ts#L339-L400)

```typescript
static makeUniforms() {
    return {
        // Rendering parameters
        renderSize: { value: new THREE.Vector2() },
        near: { value: 0.1 },
        far: { value: 1000.0 },
        numSplats: { value: 0 },
        
        // Transformation
        renderToViewQuat: { value: new THREE.Quaternion() },
        renderToViewPos: { value: new THREE.Vector3() },
        
        // Quality settings
        maxStdDev: { value: Math.sqrt(8) },
        minPixelRadius: { value: 0.0 },
        maxPixelRadius: { value: 512.0 },
        minAlpha: { value: 0.5 / 255.0 },
        
        // Anti-aliasing
        preBlurAmount: { value: 0.0 },
        blurAmount: { value: 0.3 },
        
        // Depth of field
        focalDistance: { value: 0.0 },
        apertureAngle: { value: 0.0 },
        
        // Culling and clipping
        clipXY: { value: 1.4 },
        falloff: { value: 1.0 },
        
        // Data
        packedSplats: { value: PackedSplats.getEmpty() },
        rgbMinMaxLnScaleMinMax: { value: new THREE.Vector4() },
        
        // Time
        time: { value: 0 },
        deltaTime: { value: 0 },
    };
}
```

---

## 10. Key Rendering Concepts

### A. 3D to 2D Projection
The core algorithm is **projecting a 3D Gaussian ellipsoid onto the 2D screen**:

1. **3D Covariance**: `Σ = R·S·Sᵀ·Rᵀ` (rotation + scale)
2. **Jacobian**: `J = ∂screen/∂world` (projection derivative)
3. **2D Covariance**: `Σ' = J·Σ·Jᵀ` (projected to screen)

**Reference**: [splatVertex.glsl Lines 122-167](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatVertex.glsl#L122-L167)

### B. Alpha Blending
Splats are rendered **back-to-front** with standard alpha blending:
```
C_out = C_splat·α + C_bg·(1-α)
```

Depth sorting ensures correct composition.

### C. Anti-Aliasing
Adding blur to the 2D covariance with opacity correction:
```glsl
Σ' += blur·I
α' = α·sqrt(det(Σ)/det(Σ'))
```

Prevents aliasing while conserving energy.

**Reference**: [splatVertex.glsl Lines 167-186](https://github.com/sparkjsdev/spark/blob/main/src/shaders/splatVertex.glsl#L167-L186)

---

## 11. Performance Optimizations

### Memory Efficiency
- **16 bytes per splat** (vs 60+ in uncompressed formats)
- Half-precision floats for positions
- Quantized scales (log encoding)
- Packed textures (2048×2048×depth)

### GPU Optimizations
- Instanced rendering (1 draw call for all splats)
- Early frustum culling in vertex shader
- Texture caching and reuse
- Stochastic rendering option for solid objects

### Sorting Strategies
**File**: [`src/SparkRenderer.ts`](https://github.com/sparkjsdev/spark/blob/main/src/SparkRenderer.ts)

- Bin-based approximate sorting
- Incremental updates based on camera movement
- Configurable `originDistance` threshold

---

## Additional Resources

### Official Documentation
- **Main Site**: https://sparkjs.dev/
- **Getting Started**: https://sparkjs.dev/docs/
- **System Design**: https://sparkjs.dev/docs/system-design/
- **API Reference**: https://sparkjs.dev/docs/spark-renderer/

### Example Code
- **Basic Examples**: https://sparkjs.dev/examples/
- **Source Examples**: https://github.com/sparkjsdev/spark/tree/main/examples

### Academic Papers
- **3D Gaussian Splatting**: [3D Gaussian Splatting for Real-Time Radiance Field Rendering](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/)
- **Original Paper**: Kerbl et al., SIGGRAPH 2023

---

## Summary Flow Chart

```
PLY File
   ↓
SplatLoader → Parse data
   ↓
PackedSplats → Encode to 16 bytes/splat
   ↓
DataArrayTexture → Upload to GPU
   ↓
Vertex Shader:
   1. Fetch packed data
   2. Transform to view space
   3. Build 3D covariance (R·S·Sᵀ·Rᵀ)
   4. Project to 2D (J·Σ·Jᵀ)
   5. Apply anti-aliasing blur
   6. Compute ellipse axes (eigenvalues)
   7. Generate quad vertices
   ↓
Fragment Shader:
   1. Compute distance from center
   2. Apply Gaussian falloff exp(-0.5·z²)
   3. Discard if outside bounds
   4. Output RGBA
   ↓
Alpha Blending (back-to-front)
   ↓
Final Image
```

---

**Last Updated**: November 13, 2025
**Spark Version**: v0.1.10
**Author**: Based on Spark.js source code analysis
