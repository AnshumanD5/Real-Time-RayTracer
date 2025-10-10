# Real-Time Raytracer

A high-performance, interactive CPU-based raytracer built in C++ using the Walnut framework (Vulkan + Dear ImGui). Features real-time ray tracing with progressive accumulation, physically-based materials, and interactive scene editing.

![Raytracer Demo](demo/Raytracing-demo.gif)

## Table of Contents
1. [Technical Overview](#technical-overview)
2. [Core Components](#core-components)
3. [Ray Tracing Pipeline](#ray-tracing-pipeline)
4. [Mathematical Implementation](#mathematical-implementation)
5. [Performance Optimizations](#performance-optimizations)
6. [Build Instructions](#build-instructions)
7. [Usage](#usage)

## Technical Overview

Ray tracing is a rendering technique used to generate highly realistic images by simulating the physical behavior of light as it interacts with objects in a 3D scene. It works by tracing the path of rays from the camera through each pixel on the screen and calculating how these rays intersect with scene geometry. Upon hitting a surface, additional rays may be spawned to simulate reflection and shadowing, allowing for the accurate depiction of lighting effects. Unlike rasterization, which approximates visibility and lighting, ray tracing computes light transport based on geometric and material data, producing photorealistic results at the cost of higher computational complexity.

This raytracer implements computer graphics concepts including ray-object intersection, recursive ray casting, denoising, and physically-based material properties. Built on the Walnut framework, it uses Vulkan for the backend and Dear ImGui as an immediate-mode user interface. Note that this raytracer is CPU based, for simplicity. I plan to port it to the GPU in the future, which is the intended hardware for an application like this.

The project leverages the **Walnut Framework**, which provides:
- Vulkan-based rendering backend (Image creation and GPU texture management)
- Dear ImGui integration for immediate-mode GUI (useful for performance stats and making real-time changes to the scene)
- Cross-platform window management and input handling
- Provides GLFW and GLM (OpenGL's math library)

## Core Components

### 1. Renderer Class
The renderer class is responsible for:
- **Ray Generation**: Creating primary rays from camera
- **Ray Tracing**: Implementing the core raytracing algorithm
- **Accumulation Buffer**: Managing progressive refinement
- **Color Processing**: Converting floating-point colors to RGBA format

```cpp
class Renderer {
private:
    std::shared_ptr<Walnut::Image> m_FinalImage;
    uint32_t* m_ImageData;           // Final pixel data
    glm::vec4* m_AccumulationData;   // Accumulation buffer
    uint32_t m_FrameIndex = 1;       // Progressive accumulation counter
};
```

### 2. Scene Management
The scene consists of:
- **Spheres**: Geometric primitives with position, radius, and material index
- **Materials**: PBR materials with albedo and roughness properties
- **Camera**: View frustum and ray generation parameters

### 3. Material System
Physically-based material properties:
```cpp
struct Material {
    glm::vec3 Albedo;    // Base color/reflectance
    float Roughness;     // Surface roughness (0.0 = mirror, 1.0 = diffuse)
};
```

## Ray Tracing Pipeline

![Ray tracing showing camera, view ray, light source, shadow ray, and scene object](images/raytracing-diagram.png "Ray Tracing Pipeline")


The ray tracing process begins with generating rays for each pixel on the screen. Starting from the camera position, I calculate ray directions using a pre-computed buffer that maps screen coordinates to 3D directions in world space. To clarify, the camera system precomputes ray directions per pixel based on field-of-view and aspect ratio. When the user moves or rotates the camera, it recalculates these directions to avoid recomputing projection math during rendering. Next, each ray carries an origin point and a normalized direction vector that determines where it travels through the scene.

When a ray encounters geometry in the scene, the intersection testing phase kicks in. For sphere primitives, I use the classic quadratic equation approach where the ray equation is substituted into the sphere's mathematical definition. The discriminant tells me whether an intersection occurs, and if multiple spheres are hit, I sort by distance to find the closest surface. This ensures proper depth ordering and realistic occlusion behavior.

Once a valid intersection point is found, the shading system takes over. Surface normals are computed by normalizing the vector from the sphere center to the hit point. The lighting calculation uses a simple directional light model with dot product evaluation, though the material system adds complexity through albedo and roughness parameters that influence how light bounces off surfaces.

The real magic happens during recursive ray bouncing. When a ray hits a surface, I spawn additional reflection rays based on the material properties. Rougher materials scatter rays more randomly, while smooth surfaces produce mirror-like reflections. Each bounce carries less 'energy' than the previous one, simulating how light naturally loses intensity as it bounces around a scene. I limit this to 5 bounces to prevent infinite recursion while still capturing complex lighting interactions.

## Mathematical Implementation

### Ray-Sphere Intersection Mathematics

We want to find out if a **ray** intersects a **sphere**, and if so, where.

### Ray
A **ray** is defined by:
```
r(t) = o + td
```
where:
- `o` = ray origin
- `d` = ray direction (normalized or not)
- `t` = parameter (distance along the ray)

### Sphere
A **sphere** is defined by:
```
||p - c||² = R²
```
where:
- `p` = point on the sphere
- `c` = sphere center
- `R` = radius

We want to solve for **t** such that the ray hits the sphere.

### Substituting the Ray into the Sphere Equation

Substitute `r(t)` into the sphere's equation:
```
||o + td - c||² = R²
```

Let:
```
oc = o - c
```

Then expand the squared norm:
```
(oc + td) · (oc + td) = R²
```

Expanding using dot product properties:
```
(oc · oc) + 2t(oc · d) + t²(d · d) = R²
```

Rearranged:
```
t²(d · d) + 2t(oc · d) + (oc · oc - R²) = 0
```

This is a **quadratic equation** in terms of `t`:
```
at² + bt + c = 0
```

Where:
- `a = d · d`
- `b = 2(oc · d)`
- `c = oc · oc - R²`

### Implementation

From my `TraceRay()` function:

```cpp
glm::vec3 origin = ray.Origin - sphere.Position;
float a = glm::dot(ray.Direction, ray.Direction);           // d·d
float b = 2.0f * glm::dot(origin, ray.Direction);           // 2(oc·d)
float c = glm::dot(origin, origin) - sphere.Radius * sphere.Radius; // oc·oc - R²

float discriminant = b * b - 4.0f * a * c;
```

### Solving the Quadratic

If `discriminant < 0`, no real solutions → **miss**.

If `discriminant ≥ 0`, solve using the quadratic formula:
```
t = (-b ± √(b² - 4ac)) / (2a)
```

You pick the smaller (closer) solution for the **first point of intersection**:

```cpp
float closestT = (-b - glm::sqrt(discriminant)) / (2.0f * a);
```

This gives the first point of intersection along the ray.

### Progressive Accumulation Algorithm

**Temporal Anti-Aliasing through Frame Accumulation**:
```cpp
// Accumulate color samples
m_AccumulationData[pixelIndex] += currentFrameColor;

// Compute average
glm::vec4 finalColor = m_AccumulationData[pixelIndex] / (float)m_FrameIndex;
```

This technique reduces noise and improves image quality over time by averaging multiple samples per pixel. Basically, the image converges towards its most "accurate self" as more samples come in.

## Performance Optimizations

### Multi-threading
The code includes parallel execution:
```cpp
#define MT 0  // Multi-threading toggle
#if MT
std::for_each(std::execution::par, m_ImageVerticalIter.begin(), m_ImageVerticalIter.end(), 
    [this](uint32_t y) { /* Parallel row processing */ });
#endif
```

The most significant optimization is the multi-threading system that distributes pixel calculations across all available CPU cores. Rather than processing one pixel at a time, the renderer can tackle entire rows simultaneously, dramatically reducing render times on modern multi-core processors.

Memory management plays a crucial role in performance as well. I use contiguous memory layouts for the pixel buffer, which improves cache efficiency since the CPU can predict and preload nearby memory addresses. When the viewport changes size, the dynamic resizing system efficiently reallocates buffers without unnecessary memory fragmentation. Pre-allocated iteration vectors eliminate the overhead of repeated memory allocations during the render loop.

Early ray termination provides another performance boost by immediately stopping rays that miss all scene geometry. Instead of continuing expensive bounce calculations, these rays simply return the sky color and break out of the ray tracing loop. This optimization becomes more valuable in scenes with lots of empty space where many rays never hit anything after their initial bounce.

## Rendering Features

The progressive refinement system is one of my favorite features. In accumulation mode, each frame contributes additional samples to the final image, progressively reducing noise and improving color accuracy. This creates a satisfying experience where the image starts rough but quickly converges to a clean one. When the camera moves or scene parameters change, the system automatically resets the frame counter to ensure the user gets immediate visual feedback rather than waiting for the accumulation to rebuild.

### Material Properties
- **Albedo**: Base surface color/reflectance
- **Roughness**: Controls reflection sharpness (perfect mirror to completely diffuse)

Through Dear ImGui panels, you can adjust sphere positions and radii in real-time, watching as shadows and reflections update instantly. Material sliders let you experiment with different surface properties. Performance monitoring shows frame times and render statistics for understanding the computational cost of different scene configurations.

### Sky/Environment Mapping
Simple procedural sky color for rays that miss all geometry:
```cpp
glm::vec3 skyColor = glm::vec3(0.6f, 0.7f, 0.9f); // Soft blue gradient
```

## Build Instructions

### Prerequisites
- **Visual Studio 2019+** or equivalent C++17 compiler
- **Vulkan SDK** for graphics API support  

### Dependencies
- **Walnut Framework**: Vulkan backend and Dear ImGui integration
- **GLM**: OpenGL Mathematics library for vector operations
- **C++17**: Modern language features and execution policies

### Running the application
1. Clone the repository in a local directory:
   ```bash
   git clone https://github.com/Anshuman-Dhillon/Real-Time-RayTracer.git
   ```

   OR just download it as a .zip then unzip it wherever you'd like

2. Navigate to the main folder (Raytracer) -> bin -> Release-windows-x86_64 -> RayTracer -> Run RayTracer.exe

## Usage

### Controls

**Mouse**: Right click and drag to orbit around the scene<br>
**WASD**: To move around inside the scene<br>
**Settings Panel**: Toggle accumulation mode and reset frame counter<br>
**Scene Panel**: Adjust object properties in real-time<br>


## Project Structure
```
RayTracer/
├── src/
│   ├── Renderer.cpp      # Core raytracing implementation
│   ├── Renderer.h        # Renderer class definition  
│   ├── WalnutApp.cpp     # Application entry point and UI
│   ├── Camera.h/.cpp     # Camera system
│   └── Scene.h           # Scene data structures
├── external/
│   └── Walnut/           # Walnut framework
└── premake5.lua          # Build configuration
```

## Possible Future Improvements

- Porting to the GPU to achieve true parallelism (could use CUDA)
- BVH acceleration structures for complex scenes
- Temporal and spatial denoising algorithms
- Advanced PBR materials with normal mapping
- Volumetric effects (fog, smoke, participating media)
- Area lights for soft shadows
- Post-processing pipeline (tone mapping, bloom, DOF)
- Asset streaming for large scenes

## References

- [*Ray Tracing in One Weekend* book series by Peter Shirley](https://raytracing.github.io/books/RayTracingInOneWeekend.html) - Foundational guide on implementing a basic CPU ray tracer from scratch
- [*Real-Time Ray Tracing — Part 1* by Gimpy Software](https://gimpysoftware.com/index.php/tutorials/58-realtime-raytracing-part-1) - Explains the fundamentals of real-time ray tracing techniques
- [*Walnut Framework* by The Cherno](https://github.com/StudioCherno/Walnut) - C++ application framework
