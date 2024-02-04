# Fundamentals of Graphics Programming

## Graphics Pipeline

In order to grasp the content of this document a basic understanding of a graphics rendering pipeline is needed. Below is an example of the Vulkan Graphics
Pipeline. You can skip this section if you know already about the graphics pipeline.

![[Unreal Engine Render Dependency Graph/Diagrams/VulkanPipeLine.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/08e9c57045b6a88b7918499961a3c2e2e23e83ff/Diagrams/VulkanPipeLine.png)

There are many types of graphics pipelines used across many platforms some examples are Vulkan, DirectX, OpenGL. All pipelines share common steps in the
pipeline such as the Input Assembler, Vertex Shader, Rasterization and Fragment Shader (also know and pixel shader). Some API’s have different steps before or
after the Rasterizer that may optimize the pipeline specific operations to that platform.

`Step 1.` The input assembler collects the raw vertex data from the buffers you specify and as far as my knowledge in combination with shader code like HLSL will
allow you to bind to input Semantics which I will explore later on.

`Step 2.` The Vertex shader is run for every vertex and generally applies transformations to turn vertex positions from model space to screen space. It also passes per-vertex data down the pipeline. In general before doing any Matrix Transformations from the vertex to world space, local space or screen space. The vertex data will be interpreted in the standard Clip Space.

![[Unreal Engine Render Dependency Graph/Diagrams/Coordinate Systems.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/08e9c57045b6a88b7918499961a3c2e2e23e83ff/Diagrams/Coordinate%20Systems.png)

`Step 3.` **The Geometry shader** is run on every primitive (triangle, line, point) and can discard it or output more primitives than came in. This is similar to the
tessellation shader, but much more flexible. However, it is not used much in today's applications, reiterating some API’s are different.

`Step 4. `**The Rasterization stage** discretizes the primitives into fragments. These are the pixel elements that they fill on the framebuffer. Any fragments that fall outside the screen are discarded and the attributes outputted by the vertex shader are interpolated across the fragments, as shown in the figure. Usually the
fragments that are behind other primitive fragments are also discarded here because of depth testing. Depth testing is used along side a depth buffer which
holds info for the culling process. From my understanding the Rasterizer is the only place in the pipeline that’s not programable.

`Step 5.` **Fragment Shader** or the **Pixel shader** as Unreal calls it. The pixel shader is invoked for every fragment that survives the rasterization stage and determines
which framebuffer(s) the fragments are written to along with which color and depth values. It can do this using the interpolated data from the vertex shader,
which can include things like texture coordinates and normals for lighting.
If interpolation is confusing imagine this example scenario between vertex shader and pixel shader.

Suppose you had two vertex values A & B. Both vertices contain a unique 2D screen position and a 3D pixel color. Now if vertex A was a point on the screen with
a color value of Red and vertex B was another point with the color value blue and you drew a line connecting A to B you’d have something like this.
Red(VA)——Purple—-Blue(B), The vertex shader would draw the line connecting A to B. Then pass that off to the Pixel shader which would interpolate the color
between A to B. Point A is purely red and Point B is purely blue giving us a middle color of purple (which is a mix of Red and blue).

`Step 6`. API specific operations can be seen before the Pixel Shader hands off the final image to the framebuffer.

### GPU Buffers

Key thing to understand is that a buffer is just a resource stored in memory on the GPU. These resources are declared on the CPU and then Mem copied to the
GPU to be used on various rendering processes. Alignment is important in most cases when defining data on the cpu before passing it of to the GPU, so be
prepared to see things about padding and alignment when interfacing with the Graphics APIs. There are so many kinds of buffers but some of the most common
buffers are listed below.

**Vertex Buffer** - Holds data that defines all vertices the input assembler will then read this and bind it to the vertex shader we specified in the GPU Pipeline.

**Index Buffer** - The Index buffer is an array of pointers to vertices in the vertex buffer, the index buffer usually reads three vertices at a time. However you can
specify offsets, this allows us to create multiple combinations from the vertex buffer to feed to the vertex shader. We can define different draw calls to use some
examples are screen space only quads, triangles or other primitives.
Command Buffer - Command buffers are used to record the commands that are required to render a frame. These commands include tasks such as setting up
the viewport, binding shaders, textures, and issuing draw calls to render the geometry. Once the command buffer is recorded, it can be submitted to the GPU for execution.

**Depth Buffer** - The depth buffer (also known as the Z-buffer) is a memory buffer that is used in 3D graphics to store the depth information for each pixel on the
screen. It is used to determine the relative depth of the objects in a scene, and to ensure that objects that are closer to the viewer are drawn on top of objects
that are further away.

**GBuffer** - The GBuffer, short for Geometry Buffer, is a set of multiple off-screen render targets that are used to store various information about the geometry in
a 3D scene. This information can include things like depth, normals, albedo (color), specular, and other data. The GBuffer is typically used in deferred shading,
which is a technique that separates the process of capturing the geometry of a scene from the process of applying lighting and shading to that geometry.

**Texture Buffer** - A texture buffer is a GPU buffer that is used to store texture data. A texture is a 2D or 3D image that is applied to the surfaces of 3D models to
add visual detail and realism to a scene. Texture buffers are used to store the pixel data for textures, which can include things like color, alpha, and normal
information.

### HLSL

In Unreal Engine shader code is written using HLSL with the file extension types .usf and .ush.

I like to imagine the HLSL like assembly code. Where you have Input registers and Output registers. You define your inputs from your vertex shader and it’s
outputs as inputs to your pixel shader. Regular semantics are used to define the meaning of input and output data for a particular stage of the pipeline. This
could include things like position, color, normal, and texture coordinates. You can call these “Regular Semantics” anything like but you will need to bind
resources (Buffers) to them using the Input Assembler as illustrated earlier.

“System semantics”, on the other hand, are used to define the meaning of input and output data that is specific to a particular platform or API. These semantics
are used to define how the data is passed between the GPU and the API, such as the DirectX or OpenGL API.

For example, you would use the "SV_VERTEXID" semantic to indicate that a particular input variable contains the vertex ID, which is a system semantic that is
specific to DirectX. This semantic is tied to a DirectX specific draw call where the SV_VERTEXID semantic is incremented based on the current vertex being
processed the vertex buffer the vertex shader is using.

Below is the Triangle HLSL code we will be binding too to draw our triangle later on.

```cpp
// TriangleVS Binable Name for entry point for a custom vertex shader
void TriangleVS(
in float2 InPosition : ATTRIBUTE0, // First Input Bindable Regular Symantic
in float4 InColor : ATTRIBUTE1, // Second Input Bindable Regular Symantic
out float4 OutPosition : SV_POSITION, // System Symantic Ouput position to Pixel Shader
out float4 OutColor : COLOR0 // System Symantic Output Color to Pixel Shader
)
{
OutPosition = float4(InPosition, 0, 1);
OutColor = InColor;
}
// TrianglePS Bindable entry point for a custom Pixel Shader
void TrianglePS(
in float4 InPosition : SV_POSITION, //System Symantic Input position to Pixel Shader
in float4 InColor : COLOR0, // System Symantic Input Color to Pixel Shader
{
out float4 OutColor : SV_Target0) // System Symantic Render Target, IE. The 2D texture resource Render to.
OutColor = InColor;
}
```
