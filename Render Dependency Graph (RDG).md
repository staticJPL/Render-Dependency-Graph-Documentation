# Render Dependency Graph (RDG)

In 2017 Yuriy O’Donnell pioneered a render graph system while working for Frostbite and presented the first Frame Graph at GDC. Consequently the series of
advantages this system provided was taken into Unreal Engine and as of 2021 the use of the render graph has become the standard in AAA game engine
development.

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_CommandQueue.png]]

### Properties

With the Render Graph we are able to abstract render operations to a single way to produce render code. This allows for clarity of the code and debuggability
for tools to interpret resource lifetimes and render pass dependencies to effectively reduce development time.

The next generation of Graphics APIs such as DX12 and Vulkan manage resource states and transitions depending on operations we want to perform.

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_ResourceTransition.png]]

Using render graphs these operations can be handled automatically without manual input. As seen in the diagram above, a graphics programmer can declare
what resources are needed for their shader input, render targets and or depth-stencils. Resource transitions are handled by the graph, you can visualize the
green lines as “read” and red lines as “write” operations. Each render graph node has knowledge of these interactions and allows it to place `Barriers` for
resource transitions. This means that a optimal barrier ensures that there is an optimal command queue setup. So if `resource A` is used as a shader resource for
`Pass 1` but as a render target for `pass 2` then we will still need a resource transition to render between the two targets. This reduces calls and saves memory
allocation.

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_PassResourceLifetime.png]]

In the image above, for example, `resource A` is used only up to the third pass. On the other hand, `resource C` starts getting used in the fourth pass, and so its
lifetime does not overlap the one of `resource A`, meaning that we can use the same memory for both resources.

The same concept applies for resource A and D, and in general we will have multiple ways to overlap our memory allocations, so we will also need clever ways to detect the best allocation strategy.


### RDG Resources

There are resources that are used on “Per-frame” basis: which are technically called graph or transient resources since their lifetime can be fully handled by the
render graph. Some examples of “Per-Frame” resources are `Gbuffers` and Camera Depth which are deferred in lighting pass.

`Transient` resources are meant for a render graph to last for a specific time within a single frame so there is a lot of potential for memory re-use. The are other
resources that are used outside and are dependent on other resources like a window swapchain back buffer, so the graph in this case will limit itself to just
managing their state. This is know as `external resources`.

### Transient Resource System

The lifetime of transient resources can have what’s called “resource aliasing” for these resources (according to DX12 terminology).

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_Aliasing.png]]

Aliased resources can spare more than 50% of the used resource allocation space, especially when using a render graph. They add an additional managing
resource complexity to the scene, but if we want to spare memory, they are almost always worth it.

### Build Cross-Queues Synchronization

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_CommandDependencyTree.png]]


Lastly the graph allows us to use multiple command queues and run them in parallel, using a dependency tree we can synchronize these mechanism to prevent
race conditions on the shared resources.

An acyclic graph of render passes, which we have after we lay down every pass from the dependent queues. Each level of a dependency tree, called dependency
level, will contain passes independent of each other, in the sense of their resource usage. By doing that we can ensure that every pass in the same dependency
level can potentially run asynchronously. We can still have cases of multiple passes belonging to the same queue in the same dependency level, but this does
not bother us.

As a consequence, we can put a synchronization point with a GPU fence at the end of every dependency level: this will execute the needed resource transitions,
for Every queue on a single graphics command list. This approach of course does not come for free, since using fences and syncing different command queues
has a time cost. In addition to that, it will not always be the optimal and smallest amount of synchronizations, but it will produce acceptable performance and it
should cover all the possible edge cases.

Thus this highlights the advantages of the Render Graph in a nutshell. Better resource management, easier debugging tools and parallel command list
synchronization. Additional information on this is linked in the references.

### RDG Dynamics

Some new terminology needs to be addressed on top of what we already seen in the context of RDG.

- **View**: a single “viewport” looking at the FScene. When playing in split screen or rendering left and right eye in VR for example, we are going to have two views.

- **Vertex Factory**: class encapsulating vertex data and it is linked to the input of a vertex shader. We have different vertex factory types depending on the kind of mesh we are rendering.

- **Pooled Resource**: graphics resource created and handled by the RDG. Their availability is guaranteed during RDG passes execution only.

- **External Resource**: graphics resource created independently from the RDG. 

The workflow of Unreal Engines RDG can be Identified in a 3 step process:

**Setup phase**: declares which render passes will exist and what resources will be accessed by them.

**Compile phase**: figure out resources lifetime and make resource allocations accordingly.

**Running/Execute phase**: all the graph nodes get executed.

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_Stages.png]]

### Setup Stage

The setup stage begins inside` FRenderModule`, which is triggered by only the render thread main function. Which builds passes for the visible views and all
objects associated with them.

```cpp
FDeferredShadingSceneRenderer::Render(FRHICommandListImmediate& RHICmdList)
```

The generic syntax for all RDG Passes will be created in this Generic Case

```cpp
// Instantiate the resources we need for our pass
FShaderParameterStruct* PassParameters = GraphBuilder.AllocParameters&lt;FShaderParameterStruct>();
// Fill in the pass parameters
PassParameters->MyParameter = GraphBuilder.CreateSomeResource(MyResourceDescription, TEXT("MyResourceName"));
// Define pass and add it to the RDG builder
GraphBuilder.AddPass( RDG_EVENT_NAME("MyRDGPassName"), PassParameters, ERDGPassFlags::
Raster, [PassParameters, OtherDataToCapture](FRHICommandList&RHICmdList) {
// … pass logic here, render something! ...
}
```

- **Pass Name**: This will ultimately be represented by an object of type FRDGEventName containing the description of the pass. It is used for debugging and profiling tools.

- **Pass Parameters**: This object is expected to derive from a Shader Parameter Struct which has to be created with GraphBuilder.AllocParameters() and needs to be defined with the macro `BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters, `) The PassParameters will need to distinguish between at least a shader resources and render targets in order to detect proper transitions. (more on this subject in the Shader Parameters) and it can come from either:
	- A shader uniform buffer proper to a shader (e.g. in the case we have only one shader like a compute shader) and so in this case the PassParameters will be of type FMyShader::FParameters.
	- A generically defined shader uniform buffer, which will usually be defined in the source (.cpp) file. The usual name of these buffers will include“PassParameters” to specify that they are used for a whole pass instead of a single shader.

- **Pass Flags**: Set of flags of type ERDGPassFlags, they are mainly used to specify the kind of operations we will be doing inside the pass, e.g. raster, copy and compute.

- **Lambda Function**: This will contain the “body” of our pass, and so the logic to execute at run time. With the lambda we can capture any number of objects we want to later use to set up our rendering operations. Remember, after collection, they are not executed immediately and will be delayed but there are situations where it can be immediate.

### Compile Phase

The `compile` phase is completely autonomous and a “non-programmable” stage, in the sense that the render pass programmer does not have influence on it.
In this phase the graph gets inspected to find all the possible flow optimizations, it will:

1. Exclude unreferenced but defined resources and passes: if we want to draw a second debug view of the scene, we might be interested to draw only certain passes for it.
2. Compute and handle used resources lifetime.
3. Resources Allocation.
4. Build optimized resource transition graph.

### Running Stage

With the term Running Stage we mean the time when the lambda function of an RDG pass gets executed. This will happen asynchronously and the exact
moment is completely up to the RDG.

When the lambda body executes, the available input will be the variables captured by the lambda and a command list (either `RHIComputeCommandList&` for
`Compute `/ `AsyncCompute` workloads or `FRHICommandList&` for raster operations).
What essentially happens inside the lambda body is the following


- Set Pipeline state object: e.g. setting rasterizer, blend and depth/stencil states.
- Set Shaders and their Attributes: select what shaders to use, bind them to the current pipeline and set parameters for them, which means binding resources to the shader slots, on the current command list.
- Send copy/draw/dispatch command: send render commands on the command list.

### Execute Phase

Execution phase is as simple as navigating through all the passes that survived the compile phase culling and executing the draw and dispatch commands on the
list. Up until the execute phase all the resources were handled by opaque and abstract references, while at execute phase we access the real GPU API resources
and set them in the pipeline. The preparation of command lists, on the CPU side, can be potentially parallelized quite a lot: in most of the cases, each pass
command list setup is independent from each other. Aside from that, command list submissions on a single command queue is not thread safe, and in any case
we would first need to determine if adding parallelization would bring significant gains.

#### Shader Types

The base class for shaders is FShader, but we find two main types of shaders that we can use:
- FGlobalShader: all the shaders deriving from it are part of the global shaders group. A global shader produces a single instance across the engine, and it can only use global parameters.
- FMaterialShader: all the derived classes are the ones that use parameters tied to materials. If they also use parameters tied to the vertex factory, then the class to refer to is FMeshMaterialShader.

Note* I’ll be using the Global Shader since I rendered inside the post processing pass. The Material shaders are heavily tied to a vertex factory, which I didn’t
have time to cover and or experiment with. I will provide some resources for that in the reference section if you’re interested.

### Shader Parameters

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_ShaderParameterBinding.png]]

The shader parameters are the objects that are gonna identify the resource slots used by a shader. These parameters are used when setting resources for a
graphics compute or computer operation.

The process of setting a shader parameter will consist in binding a resource to the command list at the index specified by the shader parameter.

**Note*** if binding and context is confusing, this is because an understanding a lower level GPU and graphics API knowledge is probably missing. To address this
I’ve given a brief explanation in the supplemental explanation section.

We have the following types of Shader Parameters, as seen in `ShaderParametersUtils.h` and `ShaderParameters.h`:

- **FShaderParameter**: shader parameter’s register binding. e.g. float1/2/3/4, can be an array, UAV.
  
- **FShaderResourceParameter**: shader resource binding (textures or samplerstates).
  
- **FRWShaderParameter**: class that binds either a UAV or SRV resource.

- **TShaderUniformBufferParameter**: shader uniform buffer binding with a specific structure (templated). This parameter references a struct which contains all the resources defined for a specific shader. More info later.

As many cases in Unreal Engine, shader parameter classes use macros to define how they are composed. The most important part of the shader parameters is its
Layout, an internal variable defined at compile time that specifies its structure, composed of `Layout Fields`.

```cpp
LAYOUT_FIELD(MyDataType, MyFiledName);
LAYOUT_FIELD(FShaderResourceParameter, UAVParameter);
```

The way the layout will be used depends on the type of shader parameters and it can contain any data (e.g. a parameter index). Its purpose is always to hold
information about shader parameters (e.g. CBVs, SRVs, UAVs in D3D12) so that we can use them to bind to resources at the moment of executing the shader in
the command list.

### Shader Uniform Buffer Parameter

The concept of `Uniform Buffer Parameter` in Unreal Engine is very different from what we are used to in standard computer graphics: here it is essentially
defined as a struct of shader parameters.

Uniform Buffers, as previously mentioned in the RDG chapter, can be defined using a Shader Parameter Struct macro, either inside a shader class declaration or
in global scope.

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters, )
	SHADER_PARAMETER_RDG_TEXTURE(Texture2D, InputTexture)
	SHADER_PARAMETER_SAMPLER(SamplerState, InputSampler)
	RENDER_TARGET_BINDING_SLOTS()
END_SHADER_PARAMETER_STRUCT()
```

This family of macros is very flexible and it can contain:

- **Shader Parameters**: textures, samplers, buffers and descriptors. For a full list of macros reference to ShaderParameterMacros.h.
- **Nested Structs**: we can encapsulate the definition of a shader parameter struct into another. This is achieved using the macro `SHADER_PARAMETER_STRUCT(StructType,MemberName)` and `SHADER_PARAMETER_STRUCT_ARRAY(..)`.`
- **Binding Slots**: Available with the macro `RENDER_TARGET_BINDING_SLOTS()` which adds an array of assignable Render Targets to the parameter struct we use.

**Usage**
- If defined outside a shader, the name “FMyShaderParameters” will usually be `F<MyPassName>` PassParameters. These shader parameter structs are used as Pass Parameters for the RDG, as described in the RDG section of this article.
- If defined inside a shader class, the name “FMyShaderParameters” will usually be `FParameters`. We will also need to use this macro at the top of the shader class `SHADER_USE_PARAMETER_STRUCT(FMyShaderClass, FBaseClassFromWhatMyShaderDerivesFrom)`.

**Set Shader Parameters**

Most of the times when using a shader inside an RDG pass you can call
```cpp
SetShaderParameters(TRHICmdList& RHICmdList, const TShaderRef&lt;TShaderClass>& Shader, TShaderRHI* ShadeRHI, const typename
TShaderClass::FParameters& Parameters)
```

![[Unreal Engine Render Dependency Graph/Diagrams/RDG_SettingShaderParams.png]]

from ShaderParameterStruct.h will be called to bind input resources to a specific shader.

The function will first call `ValidateShaderParameters(Shader, Parameters);` to check that all the input shader resources cover all the expected shader parameters.
Then it will start to bind all the resources to the relative parameters: every parameter type listed at the beginning of this section (e.g. FShaderParameter,
FShaderResourceParameter, etc.) will have their own call for getting bound to the command list. A scheme of when we set a uniform buffer resource is as follows:

**BufferIndex** is used for all the `FParameterStructReference`, but also for the basic `FParameters` elements, since they are stored in buffers as well.
What happens inside CmdList::SetUniformBuffer with such input parameters is completely up to the render platform we are using and it varies a lot from case to
case.

### Shader Macro Usage & Setup

**Local Parameters**

Starting with some Parameters, if we want to generate our own Uniform Buffer (Constant Buffer used by many shaders). Let’s show an example between HLSL
declarations and our Macro Setup

```cpp
float2 ViewPortSize;
float4 Hello;
float World;
float3 FooBarArray[16];
Texture2D BlueNoiseTexture;
SamplerState BlueNoiseSampler;
// Note Sampler States are objects that we used to "Sample a texture" basically reading
// the sample if we want to do masking, blending or other render wizardry.
Texture2D SceneColorTexture;
SamplerState SceneColorSampler;
RWTexture2D<float4> SceneColorOutput;
```

as explained in the previous section we need to use an internal macro so we can bind these parameters.

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters,)
	SHADER_PARAMETER(FVector2f,ViewPortSize)
	SHADER_PARAMETER(FVector4f,Hello)
	SHADER_PARAMETER(float,World)
	SHADER_PARAMETER(FVector3f,FooBarArray,[16])
	SHADER_PARAMETER_TEXTURE(Texture2D,BlueNoiseTexture)
	SHADER_PARAMETER_TEXTURE(SamplerState,BlueNoiseSampler)
	SHADER_PARAMETER_TEXTURE(Texture2D,SceneColorTexture)
	SHADER_PARAMETER_TEXTURE(SamplerState,SceneColorSampler)
	SHADER_PARAMETER_UAV(Texture2D,SceneColorTexture)
END_SHADER_PARAMETER_STRUCT()
```

The macro `SHADER_PARAMETER_STRUCT` will fill in all the data internally to generate reflective data at compile time.

```cpp
const FShaderParametersMetadata* ParametersMetadata = FShaderParameters::FTypeInfo::GetStructMetadata();
```

**Alignment Requirements**

You need to conform to alignment. Unreal Adopts the principle of 16-byte automatic alignment, thus the order of any members matters when declaring that
struct.

The main rule is that each member is aligned to the next power of its size, but only if it larger that 4 Bytes. For example:

Pointers are 8-byte aligned
- float,uint32,int32 are 4-byte aligned
- FVector2f, FIntPoint is 8-byte aligned
- FVector and FVector4f are 16-byte aligned

if you don't follow this alignment engine with throw an assert statement at compile time.

Automatic alignment of each member will be inevitably create padding, as indicated below:

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters,)
	SHADER_PARAMETER(FVector2f,ViewPortSize) // 2 x 4 bytess
	// 2 x 4 byte of padding
	SHADER_PARAMETER(FVector4f,Hello) // 4 x 4 bytes
	SHADER_PARAMETER(float,World) // 1 x 4 bytes
	// 3 x 4 byte of padding
	SHADER_PARAMETER(FVector3f,FooBarArray,[16]) // 4 x 4 x 16 bytes
	SHADER_PARAMETER_TEXTURE(Texture2D,BlueNoiseTexture) // 8 bytes
	SHADER_PARAMETER_TEXTURE(SamplerState,BlueNoiseSampler) // 8 bytes
	SHADER_PARAMETER_TEXTURE(Texture2D,SceneColorTexture) // 8 bytes
	SHADER_PARAMETER_TEXTURE(SamplerState,SceneColorSampler) // 8 bytes
	SHADER_PARAMETER_UAV(Texture2D,SceneColorTexture) // 8 bytes
END_SHADER_PARAMETER_STRUCT()
```

```cpp
SHADER_PARAMETER(FVector3f,WorldPositionAndRadius) // DONT DO THIS
// ----------------------- //
// Do this
SHADER_PARAMETER(FVector, WorldPosition) // Good
SHADER_PARAMETER(float,WorldRadius) // Good
SHADER_PARAMETER_ARRAY(FVector4f,WorldPositionRadius,[16]) // Good
```

**Binding the shader**

After we’ve setup the shader parameters and alignment is all good declare it with

```cpp
SHADER_USE_PARAMETER_STRUCT(FMyShaderCS,FGlobalShader)
```

```cpp
class FMyShaderCS : public FGlobalShader
{
DECLARE_GLOBAL_SHADER(FMyShaderCS);
SHADER_USE_PARAMETER_STRUCT(FMyShaderCS,FGlobalShader)
static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters) {
	return true;
}
	using FParameters = FMyShaderParameters;
}
// Additionally we could inline the struct definition like this
class FMyShaderCS : public FGlobalShader
{
	DECLARE_GLOBAL_SHADER(FMyShaderCS);
	SHADER_USE_PARAMETER_STRUCT(FMyShaderCS,FGlobalShader)
	static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters) 
	{
		return true;
	}
	using FParameters = FMyShaderParameters;
BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters,)
	SHADER_PARAMETER(FVector2f,ViewPortSize) // 2 x 4 bytess
	// 2 x 4 byte of padding
	SHADER_PARAMETER(FVector4f,Hello) // 4 x 4 bytes
	SHADER_PARAMETER(float,World) // 1 x 4 bytes
	// 3 x 4 byte of padding
	SHADER_PARAMETER(FVector3f,FooBarArray,[16]) // 4 x 4 x 16 bytes
	SHADER_PARAMETER_TEXTURE(Texture2D,BlueNoiseTexture) // 8 bytes
	SHADER_PARAMETER_TEXTURE(SamplerState,BlueNoiseSampler) // 8 bytes
	SHADER_PARAMETER_TEXTURE(Texture2D,SceneColorTexture) // 8 bytes
	SHADER_PARAMETER_TEXTURE(SamplerState,SceneColorSampler) // 8 bytes
	SHADER_PARAMETER_UAV(Texture2D,SceneColorTexture) // 8 bytes
END_SHADER_PARAMETER_STRUCT()
}
```

```cpp
// Setting the Parameters in the C++ code before passing it off to the Lambda Function
FMyShaderParameters* PassParameters = GraphBuilder.AllocParameters<FMyShaderParameters>();

PassParameters.ViewPortSize = View.ViewRect.Size();
PassParameters.World = 1.0f;
PassParameters.FooBarArray[4] = FVector(1.0f,0.5f,0.5f);

// Get access to the shader we class we declared Global
TShaderMapRef<FMyShaderCS> ComputeShader(View.Shadermap);
RHICmdList.SetComputeShader(ShaderRHI);

// Setup your Shader Parameters
SetShaderParameters(RHICmdList,*ComputeShader,ComputeShader->GetComputeShader(),Parameters);
RHICmdList.DispatchComputeShader(GroupCount.X,GroupCount.Y,GroupCount.Z);
```

**Global Uniform Buffer**

Process is the same with some small differences

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FSceneTextureUniformParameters,/*Blah_API*/)
	// Scene Color / Depth
	SHADER_PARAMETER_TEXTURE(Texture2D, SceneColorTexture)
	SHADER_PARAMETER_SAMPLER(SamplerState,SceneColorTextureSampler)
	SHADER_PARAMETER_TEXTURE(Texture2D, SceneColorTexture)
	SHADER_PARAMETER_SAMPLER(SamplerState,SceneDepthTextureSampler)
	SHADER_PARAMETER_TEXTURE(Texture2D<float>, SceneDepthTexturesNonMS)
	//GBuffer
	SHADER_PARAMETER_TEXTURE(Texture2D, GBufferATexture)
	SHADER_PARAMETER_TEXTURE(Texture2D, GBufferBTexture)
	SHADER_PARAMETER_TEXTURE(Texture2D, GBufferCTexture)
	SHADER_PARAMETER_TEXTURE(Texture2D, GBufferDTexture)
	SHADER_PARAMETER_TEXTURE(Texture2D, GBufferETexture)
	SHADER_PARAMETER_TEXTURE(Texture2D, GBufferVelocityTexture)
	// ...
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

After the struct setup you need to call the implement macro, following the string is the real name defined in the HLSL file.

```cpp
IMPLEMENT_GLOBAL_SHADER_PARAMETER_STRUCT(FSceneTexturesUniformParameters,"SceneTextureStruct");
```

Now inside the Unreal system, Common.ush will refer to the code generated, you will see this Common.ush get included in a lot of HLSL files. There are other
includes that you can use that have some useful functions for render code.

Now the uniform buffer we set can be accessed anywhere
```cpp
// Generated file that contains the unifrom buffer declarations that we need to compile the shader want
#include "/Engine/Generated/GeneratedUniformBuffers.ush"
```

![[Unreal Engine Render Dependency Graph/Diagrams/SceneTextureStruct.png]]

Now reference our uniform buffer inside our Parameter struct

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FParameters,)
	//...
	// Here we ref our unifor buffer
	SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters,ViewUniformBuffer)
END_SHADER_PARAMETER_STRUCT()
```

Again setup the parameter in C++ before passing into the Lambda Function

```cpp
FMyShaderParameters* PassParameters = GraphBuilder.AllocParameters<FDMyShaderParameters>();
PassParameters.ViewPortSize = View.ViewRect.Size();
PassParameters.World = 1.0f;
PassParameters.FooBarArray[4] = FVector(1.0f,0.5f,0.5f);
PassParameters.ViewUniformBuffer = View.ViewUniformBuffer;
```

