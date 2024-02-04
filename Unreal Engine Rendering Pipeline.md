# Unreal Engine Rendering Pipeline

Before getting into what the rendering pipeline of Unreal, the concept of a rendering pass and deferred rendering needs to addressed.

A `Rendering Pass `is set of one to many draw calls executed on the GPU. Usually many draw calls are grouped together to ensure proper order of execution. This
is because the output of a previous pass may be used as input for other sequential passes.

`Deferred rendering` is a default method in Unreal that renders lights and materials in a separate pass. This separate pass waits for the base pass to accumulate
the information about key information such as opacity, specular, diffusion, normals etc. An example below shows how the deferred rendering works.

![[Unreal Engine Render Dependency Graph/Diagrams/DeferredRender.png]]

Instead of computing lighting and shading for each pixel as it rasterized, Unreal uses deferred rendering to capture information about the scene’s geometry into
the “off-screen” Gbuffers. Then it’s used on a second pass to apply lighting and shading to the scene. The advantage of this is that it allows for more efficient use
of the GPU. Separation of the geometry from lighting and shading can improve performance since it allows the GPU to process large number of lights and
effects simultaneously. Additionally this allows for flexibility with dynamic lighting and complex lighting setups.

## Pass Order in Unreal Engine

These Passes may change but in general this is the order of things. I recommend downloading RenderDoc and hooking the engine to see for yourself.

**Base Pass**
- Rendering final attributes of Opaque or Masked materials to the G-Buffer
- Reading static lighting and saving it to the G-Buffer
- Applying DBuffer decals
- Applying fog
- Calculating final velocity (from packed 3D velocity)
- In forward renderer: dynamic lighting

In Deferred mode the base pass saves the properties of materials into the GBuffer as highlighted earlier and leaves it for calculation of lighting later on.

**Geometry Passes**

The Geometry pass is where the meshes get drawn and prioritized before lighting.

#### PrePass

![[Unreal Engine Render Dependency Graph/Diagrams/PrePass.png]]Early Rendering of Depth Z-Buffer, which is used to optimize out meshes bases on translucency. This also is used to optimize out meshes that are hidden behind
other meshes to cull them out (to not render them)

**HZB**
- Generates a hierarchy Z-Buffer
The HZB is used by an occlusion culling method and by screen space techniques for ambient occlusion and reflection.

**Render Velocities**
- Saves velocity of each vertex (used later by motion blur and temporal anti-aliasing)

`Velocity` is a buffer that measures the velocity of every moving vertex and saves it into the motion blur velocity buffer. The Velocity buffer compares the difference between the current frame and one frame behind to create a mask. In `Doom 2016`they use this mask to render only meshes that are not static in a
scene to optimize rendering of meshes that moved in the next frame.

**Lighting Pass**
This is the most hardcore part of the frame, especially with a lot of dynamic and shadowed light sources.
**Direct Lighting**
- Optimized lighting in forward shading
**Non-Shadowed Lights**
- Lights in deferred rendering that don’t cast shadows
**Shadowed Lights**
- Lights that obviously cast dynamic shadows
**Shadow Depths**
- Generates depth maps for shadow-casting lights

![[Unreal Engine Render Dependency Graph/Diagrams/ShadowProjection.png]]

**Shadow Projection**
- Final Rendering of Shadows

**Indirect Lighting**
![[Unreal Engine Render Dependency Graph/Diagrams/IndirectLighting.png]]
- Screen space ambient occlusion
- Decals (non-Buffer type)

**Composition After Lighting**
- Handles subsurface scattering

**Translucency and lighting**
- Renders translucent materials
- Lighting of materials that use surface forward shading.

**Reflections**
- Reading and blending reflection capture actors’ results into a full-screen reflection buffer
  
**Screen Space Reflections**
- Real-Time dynamic reflections
- Done in Post process using a screen-space ray tracing technique

**Post Processing**
The post processing is the last pass of the render pipeline and is the part of the rendering process we will draw our triangle later on.

- Depth of Field (BokehDOFRecombine)
- Temporal anti-aliasing (TemporalAA)
- Reading velocity values (VelocityFlatten)
- Motion blur (MotionBlur)
- Auto exposure (PostProcessEyeAdaptation)
- Tone mapping (Tonemapper)
- Upscaling from rendering resolution to display’s resolution (PostProcessUpscale)
