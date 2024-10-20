# Scene View Extension Triangle Shader

![[Unreal Engine Render Dependency Graph/Diagrams/Triangle Render.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/5110a92b8c25e1eab6f28e73456570649c2d0470/Diagrams/Triangle%20Render.png)

With this knowledge the next step is to put it into practice. The “Hello World” of graphics programming is to draw a basic triangle using the Vertex Shader and
Pixel Shader. Drawing a triangle in Unreal Engine encapsulates the process for rendering anything in the Engine itself.

This section will cover how to create a render pass to draw this triangle in a step by step tutorial using the Render Dependency Graph in Unreal engine.

## Plugin/Module & Shader Folder Setup

Using a Plugin or Module depends on your use case but the important thing to know is that both “Startup Module” or “Initialize” functions in the Plugin or
Module allow you to run code before Unreal Engine is fully initialized. The reason this matters is because Unreal’s Renderer is a Module. If you don’t understand
Modules and the Engine Life Cycle, I recommend doing a little reading on it to give yourself a better understanding of the runtime linking of modules. The
Renderer is linked at runtime so shader code is compiled right before the editor starts up.

**Plugin Setup**

Go ahead and create a basic C++ project in Unreal Engine, First Person shooter will suffice.
Once you’ve create the project, navigate to your folder structure and add a Shader Folder.
Here is a rough example of what it looks like

```
├── YourProjectName
………├──> YourProjectName.Build.cs
………├──> Source Folder
………└──> ...
├──> Plugins
…………├──> Your Plugin Folder
………….……..└──> Shaders (Create the Folder here)
………….…….…….└──> .usf Files go here
…………………├──> Source (Create the Folder here)
……………………………└──>Private
……………………………└──>Public
……………………………└──>YourPluginName.Build.cs
……….└──> YourPluginName.uplugin
```


Go to the PluginName.Build.cs file and ensure the dependencies are there.

```cpp
using UnrealBuildTool;

public class YourPluginName : ModuleRules
{
    public YourPluginName(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = ModuleRules.PCHUsageMode.UseExplicitOrSharedPCHs;

        PublicIncludePaths.AddRange(
            new string[] {
                // ... add public include paths required here ...
                EngineDirectory + "/Source/Runtime/Renderer/Private"
            }
        );

        PrivateIncludePaths.AddRange(
            new string[] {
                // ... add other private include paths required here ...
            }
        );

        PublicDependencyModuleNames.AddRange(
            new string[]
            {
                "Core",
                "RHI",
                "Renderer",
                "RenderCore",
                "Projects"
                // ... add other public dependencies that you statically link with here ...
            }
        );

        PrivateDependencyModuleNames.AddRange(
            new string[]
            {
                "CoreUObject",
                "Engine",
                "Slate",
                "SlateCore"
                // ... add private dependencies that you statically link with here ...
            }
        );

        DynamicallyLoadedModuleNames.AddRange(
            new string[]
            {
                // ... add any modules that your module loads dynamically here ...
            }
        );
    }
}

```

Next set the modules loading phase inside YourPluginName.uplugin to “PostConfigInit”

```cpp
{
	"FileVersion": 3,
	"Version": 1,
	"VersionName": "1.0",
	"FriendlyName": "YourPluginName",
	"Description": "",
	"Category": "Other",
	"CreatedBy": "",
	"CreatedByURL": "",
	"DocsURL": "",
	"MarketplaceURL": "",
	"SupportURL": "",
	"CanContainContent": true,
	"IsBetaVersion": false,
	"IsExperimentalVersion": false,
	"Installed": false,
	"Modules": [
		{
			"Name": "YourPluginName",
			"Type": "Runtime",
			"LoadingPhase": "PostConfigInit" // Set it Here
		}
	]
}
```

The next step is to bind the shader folder with Unreal so it can find and compile our custom shader code.

```cpp
void FYourPluginNameModule::StartupModule()
{
	// This code will execute after your module is loaded into memory; the exact timing is specified in the .uplugin file permodule
	FString BaseDir = IPluginManager::Get().FindPlugin(TEXT("YourPluginName"))->GetBaseDir();
	FString PluginShaderDir = FPaths::Combine(BaseDir, TEXT("Shaders"));
	AddShaderSourceDirectoryMapping(TEXT("/CustomShaders"), PluginShaderDir);
}
void FYourPluginNameModule::ShutdownModule()
{
// This function may be called during shutdown to clean up your module. For modules that support dynamic reloading,
// we call this function before unloading the module.
}
IMPLEMENT_MODULE(FYourPluginNameModule, YourPluginNamePlugin)
```

You need to include "Interfaces/IPluginManager.h" so you can get access to the helper functions that grabs the base directory to your plugin shader folder
directory. The `“AddShaderSourceDirectoryMapping(TEXT("/CustomShaders")`, PluginShaderDir)” line basically binds a` virtual folder` called “/CustomShaders” I
don’t think the name matters but you can name it whatever. I could be wrong.

Next go into YourPlugin Folder to add a MyViewExtensionSubSystem.cpp file and MyViewExtensionSubSystem.h inside your Private and Public folder
respectively. 

The subsystem is ideal to create a custom FSceneViewExtensionBase pointer to this is object during `PostConfigInit` phase. This allows us to draw our
triangle to the editor viewport. You could declare a` TSharedPtr<FMyViewExtension, ESPMode::ThreadSafe>` object in MyCharacter.h and instantiate it in
`BeginPlay`. So it renders the triangle after pressing play in the editor.

**Module Setup**

**MyViewExtensionSubSystem.cpp**

```cpp
// Fill out your copyright notice in the Description page of Project Settings.
#include "UViewExtensionSubSystem.h"
#include "SceneViewExtension.h"
#include "MyViewExtension.h"
void UMyViewExtensionSubSystem::Initialize(FSubsystemCollectionBase& Collection)
{
	Super::Initialize(Collection);
	// Create Shared Pointer Call
	UE_LOG(LogTemp, Warning, TEXT("View Extension SubSystem Init"));
	// This is the Pointer to the FSceneViewExenstion you will see later on
	// You need this line to run your shader.
	this->ShaderTest = FSceneViewExtensions::NewExtension<MyFViewExtension>();
}
```

**MyViewExtensionSubSystem.h**

```cpp
// Fill out your copyright notice in the Description page of Project Settings.
#pragma once
#include "CoreMinimal.h"
#include "Subsystems/EngineSubsystem.h"
#include "MyViewExtensionSystem.generated.h"
class FViewExtension;
/**
*
*/
UCLASS()
class MyViewExtensionSubSystem: public UEngineSubsystem
{
	GENERATED_BODY()
	protected:
	// Declaration of the Pointer delegate Unreal's FViewExenstion object gives us
	TSharedPtr<FViewExtension,ESPMode::ThreadSafe> ShaderTest;
	public:
	virtual void Initialize(FSubsystemCollectionBase& Collection) override;
};
```

### FSceneViewExtensionBase

This base class is important because it allows you to hook into the render pipeline and use its delegates to insert your own custom render pass. Before `5.1`, in
order to create your own custom rendering pass a plugin or module was still needed to initialize your shader folder. However without this class you will need to
do additional C++ work to add your own delegates inside the render pipeline source code for a custom rendering pass. 

In this tutorial we will be overriding a delegate function in the FSceneViewExtensionBase class to insert a pass in Post Processing phase of the render pipeline.

In Engine Source at line 411 inside PostProcessing.cpp the delegates of FSceneViewExtensionBase are added.

```cpp
// ../Engine../PostProcessing.cpp"
const auto AddAfterPass = [&](EPass InPass, FScreenPassTexture InSceneColor) -> FScreenPassTexture
{
    // In some cases (e.g. OCIO color conversion) we want View Extensions to be able to add extra custom post processing after
    // the pass.
    FAfterPassCallbackDelegateArray& PassCallbacks = PassSequence.GetAfterPassCallbacks(InPass);
    
    if (PassCallbacks.Num())
    {
        FPostProcessMaterialInputs InOutPostProcessAfterPassInputs = GetPostProcessMaterialInputs(InSceneColor);
        
        for (int32 AfterPassCallbackIndex = 0; AfterPassCallbackIndex < PassCallbacks.Num(); AfterPassCallbackIndex++)
        {
            InOutPostProcessAfterPassInputs.SetInput(EPostProcessMaterialInput::SceneColor, InSceneColor);
            
            FAfterPassCallbackDelegate& AfterPassCallback = PassCallbacks[AfterPassCallbackIndex];
            
            PassSequence.AcceptOverrideIfLastPass(InPass, InOutPostProcessAfterPassInputs.OverrideOutput,
                AfterPassCallbackIndex);
            
            InSceneColor = AfterPassCallback.Execute(GraphBuilder, View, InOutPostProcessAfterPassInputs);
        }
    }
}
```


You see at `AddAfterPass`, `FScreenPassTexture` `InSceneColor` is passing a reference of “SceneColor” to the delegates overridden.` SceneColor` is a
`FScreenPassTexture` that describes a texture paired with a viewport rect. The Scene Color texture is written too many times throughout PostProcessing pipeline
and will be the texture we will draw our triangle onto. This will be our “Render Target”.

**Class Setup**

Let’s create another class header and CPP file inside our plugin public/private source folder. Call it MyViewExtension (or whatever you like).

```cpp
#pragma once
#include "TriangleShader.h"
#include "SceneViewExtension.h"
#include "RenderResource.h"
class YOURPLUGINNAME_API FMyViewExtension : public FSceneViewExtensionBase {
	public:
	FMyViewExtension(const FAutoRegister& AutoRegister);
	//~ Begin FSceneViewExtensionBase Interface
	virtual void SetupViewFamily(FSceneViewFamily& InViewFamily) override {}
	virtual void SetupView(FSceneViewFamily& InViewFamily, FSceneView& InView) override {};
	virtual void BeginRenderViewFamily(FSceneViewFamily& InViewFamily) override;
	virtual void PreRenderViewFamily_RenderThread(FRDGBuilder& GraphBuilder, FSceneViewFamily& InViewFamily) override{};
	virtual void PreRenderView_RenderThread(FRDGBuilder& GraphBuilder, FSceneView& InView) override;
	virtual void PostRenderBasePass_RenderThread(FRHICommandListImmediate& RHICmdList, FSceneView& InView) override {};
	virtual void PrePostProcessPass_RenderThread(FRDGBuilder& GraphBuilder, const FSceneView& View, const
	FPostProcessingInputs& Inputs) override;
	virtual void SubscribeToPostProcessingPass(EPostProcessingPass Pass, FAfterPassCallbackDelegateArray& InOutPassCallbacks,
	bool bIsPassEnabled)override;
};
```

Here we have a couple delegates denoted with `_RenderThread` and other setup functions. We will be overriding `SubscribeToPostProcessingPass` only.

Let’s setup some functions in the MyViewExtension.cpp file.

```cpp
#include "ViewExtension.h"
#include "TriangleShader.h"
#include "PixelShaderUtils.h"
#include "PostProcess/PostProcessing.h"
#include "PostProcess/PostProcessMaterial.h"
#include "SceneTextureParameters.h"
#include "ShaderParameterStruct.h"
// This Line Declares the Name of your render Pass so you can see it in the render debugger
DECLARE_GPU_DRAWCALL_STAT(TrianglePass);
FMyViewExtension::FMyViewExtension(const FAutoRegister& AutoRegister) : FSceneViewExtensionBase(AutoRegister) {

}
// Begin FLensFlareScene View
FMyViewExtension::FMyViewExtension(const FAutoRegister& AutoRegister) : FViewExtension(AutoRegister) {

}
void FMyViewExtension::SubscribeToPostProcessingPass(EPostProcessingPass Pass, FAfterPassCallbackDelegateArray&
InOutPassCallbacks, bool bIsPassEnabled)
{
	if (Pass == EPostProcessingPass::Tonemap)
	{
	// Create Raw Delegate Here, see later on
	}
}
```

Inside `SubscribeToPostProcessingPass` function I want to bring attention to the if statement. The if statement uses `EPostProcessing` Enum to define where in the
Post Processing phase you want to insert a render pass. I’ve opted to do it after `Tonemap`, however you can choose other points in the pipeline defined by that
enum. For now the goal is to draw the triangle to the Scene Color since it’s available during the entire Post Process Pass.

### Setting up Global Shaders

The next step is to create two Global shaders. One being our custom Vertex shader that handles the triangle vertex data stored in the vertex buffer. The second
shader being a pixel shader which will color our triangle after the rasterizer processed our vertices.

Again, inside the Plugin source folder we create a separate cpp/header file.
TriangleShader.cpp and TriangleShader.h

**Vertex Shader Class**

```cpp
// TrangleShader.h
// Defined here so we can access it in View Extension.
BEGIN_SHADER_PARAMETER_STRUCT(FTriangleVSParams,)
//RENDER_TARGET_BINDING_SLOTS()
END_SHADER_PARAMETER_STRUCT()
class FTriangleVS : public FGlobalShader
{
	public:
	DECLARE_GLOBAL_SHADER(FTriangleVS);
	SHADER_USE_PARAMETER_STRUCT(FTriangleVS, FGlobalShader)
	using FParameters = FTriangleVSParams;
	
	static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters) {
		return true;
}
};
```

Keeping it simple we aren’t creating any new crazy macro specific buffers or resources for the vertex shader. The RDG at minimum requires the macro
declaration.

**Pixel Shader Class**

```cpp
// TrangleShader.h
BEGIN_SHADER_PARAMETER_STRUCT(FTrianglePSParams,)
	RENDER_TARGET_BINDING_SLOTS()
END_SHADER_PARAMETER_STRUCT()
class FTrianglePS: public FGlobalShader
{
	DECLARE_GLOBAL_SHADER(FTrianglePS);
	using FParameters = FTrianglePSParams;
	SHADER_USE_PARAMETER_STRUCT(FTrianglePS, FGlobalShader)
};
```


`RENDER_TARGET_BINDING_SLOTS()` is the only resource we are passing, we are binding the viewport information or Render Target to our custom shader HLSL
code.

In both classes we need to declare them as global shaders and define the local parameter struct the shader needs. In C++ “using” gives access to the type
FParamters as `FTrianglePSParams` when declaring a pointer of that type for use elsewhere.

Let’s add the Triangle HLSL code in our Shader Folder.

Go into the Shader folder and create a file called `Triangle.usf` then paste the code below and save it.

```c
#include "/Engine/Public/Platform.ush"
#include "/Engine/Private/Common.ush"
#include "/Engine/Private/ScreenPass.ush"
#include "/Engine/Private/PostProcessCommon.ush"
void TriangleVS(
	in float2 InPosition : ATTRIBUTE0,
	in float4 InColor : ATTRIBUTE1,
	out float4 OutPosition : SV_POSITION,
	out float4 OutColor : COLOR0
	)
{
	OutPosition = float4(InPosition, 0, 1);
	OutColor = InColor;
}
void TrianglePS(
	in float4 InPosition : SV_POSITION,
	in float4 InColor : COLOR0,
	out float4 OutColor : SV_Target0)
{
OutColor = InColor;
}
```

### Creating Vertex and Index Buffers

At this point we’ve introduced mostly all the classes involved in the Rendering Pass. We still need to define the resources classes for our `Vertex Buffer` and `Index
`Buffer`.`

Inside `TrangleShader.h`

Add a struct the defines our Colored Vertex

```cpp
/** The vertex data used to filter a texture. */
// TrangleShader.h
struct FColorVertex
{
public:
	FVector2f Position;
	FVector4f Color;
};
```

Add the `FVertexBuffer` class

```cpp
// TrangleShader.h
/**
* Static vertex and index buffer used for 2D screen rectangles.
*/
class FTriangleVertexBuffer : public FVertexBuffer
{
public:
	/** Initialize the RHI for this rendering resource */
	void InitRHI() override {
	TResourceArray<FColorVertex, VERTEXBUFFER_ALIGNMENT> Vertices;
	Vertices.SetNumUninitialized(3);
	Vertices[0].Position = FVector2f(0.0f,0.75f);
	Vertices[0].Color = FVector4f(1, 0, 0, 1);
	Vertices[1].Position = FVector2f(0.75,-0.75);
	Vertices[1].Color = FVector4f(0, 1, 0, 1);
	Vertices[2].Position = FVector2f(-0.75,-0.75);
	Vertices[2].Color = FVector4f(0, 0, 1, 1);
	FRHIResourceCreateInfo CreateInfo(TEXT("FScreenRectangleVertexBuffer"), &Vertices);
		VertexBufferRHI = RHICreateVertexBuffer(Vertices.GetResourceDataSize(), BUF_Static, CreateInfo);
	}
};
```

Inside `FTriangleVertexBuffer` override `InitRHI` to initialize the vertex data. In GPU programming you need to create a context so you can bind resources from the
CPU before doing a memory copy to the GPU. In order to achieve this a context must be specified holding the name of the resource, size and other properties.
Set `VertexBufferRHI` object with `RHICreateVertexBuffer` and pass the required information.

Adding the Index Buffer

```cpp
// TrangleShader.h
class FTriangleIndexBuffer : public FIndexBuffer
{
public:
	/** Initialize the RHI for this rendering resource */
	void InitRHI() override
	{
		const uint16 Indices[] = { 0, 1, 2 };
		TResourceArray<uint16, INDEXBUFFER_ALIGNMENT> IndexBuffer;
		uint32 NumIndices = UE_ARRAY_COUNT(Indices);
		IndexBuffer.AddUninitialized(NumIndices);
		FMemory::Memcpy(IndexBuffer.GetData(), Indices, NumIndices * sizeof(uint16));
		FRHIResourceCreateInfo CreateInfo(TEXT("FTriangleIndexBuffer"), &IndexBuffer);
		IndexBufferRHI = RHICreateIndexBuffer(sizeof(uint16), IndexBuffer.GetResourceDataSize(), BUF_Static, CreateInfo);
	}
};
```

Same process here, you don’t need to use a `index buffer` for something a simple as a triangle. Index buffers hold pointers to our vertex data in the vertex buffer.

Usually the index buffer will read a combination of three vertices, but you can make any combination you want. This is optimal because we can reuse vertex data
to draw a triangle, quad or other primitives depending on your use case.

Lastly add a global declaration resource for the `Vertex Buffer`, this is used to define the input layout for the` Input assembler` so we can bind the correct
attributes in the HLSL code.

```cpp
// TrangleShader.h
class FTriangleVertexDeclaration : public FRenderResource
{
public:
	FVertexDeclarationRHIRef VertexDeclarationRHI;
	/** Destructor. */
	virtual ~FTriangleVertexDeclaration() {}
	virtual void InitRHI()
	{
		FVertexDeclarationElementList Elements;
		uint16 Stride = sizeof(FColorVertex);
		Elements.Add(FVertexElement(0, STRUCT_OFFSET(FColorVertex, Position), VET_Float2, 0, Stride));
		Elements.Add(FVertexElement(0, STRUCT_OFFSET(FColorVertex, Color), VET_Float4, 1, Stride));
		VertexDeclarationRHI = PipelineStateCache::GetOrCreateVertexDeclaration(Elements);
	}
	virtual void ReleaseRHI()
	{
		VertexDeclarationRHI.SafeRelease();
	}
};
```


Again we override `InitRHI` to set our declaration information. For the input layout we define an elements object and the `stride`. In computer graphics the `stride`
refers to the number of bytes between the start of one element and the start of the next element in an array of elements stored in memory. 

For the input assembler to read the `vertex buffer` properly it needs to know the number of bytes between the start of one vertex and the start of the next vertex. 

We can use `sizeof` function to compute the space of one vertex as an offset for next vertices. You also notice `STRUCT_OFFSET` is a macro helper to determine the offset between the values defined in our struct.

Next declare the resource global and extern them so the Renderer API can see it.

```cpp
// TriangleShader.h
extern YOURPLUGIN_API TGlobalResource<FTriangleVertexBuffer> GTriangleVertexBuffer;
extern YOURPLUGIN_API TGlobalResource<FTriangleIndexBuffer> GTriangleIndexBuffer;
extern YOURPLUGIN_API TGlobalResource<FTriangleVertexDeclaration> GTriangleVertexDeclaration;
```

back in `Triangle.cpp` declare the starting points for our shaders to match the function names in HLSL.

```cpp
#include "TriangleShader.h"
#include "Shader.h"
#include "VertexFactory.h"

// Define our Vertex Shader and Pixel Shader Starting point
// This is needed for all shaders

IMPLEMENT_SHADER_TYPE(,FTriangleVS, TEXT("/CustomShaders/Triangle.usf"),TEXT("TriangleVS"),SF_Vertex);
IMPLEMENT_SHADER_TYPE(,FTrianglePS,TEXT("/CustomShaders/Triangle.usf"),TEXT("TrianglePS"),SF_Pixel);

TGlobalResource<FTriangleVertexBuffer> GTriangleVertexBuffer;
TGlobalResource<FTriangleIndexBuffer> GTriangleIndexBuffer;
TGlobalResource<FTriangleVertexDeclaration> GTriangleVertexDeclaration;
```

Lastly define our `TGlobalResources` objects.

### Adding an RDG Pass

Returning to update your MyViewExtension cpp/header files.

```cpp
// MyViewExtension.h
class YOURPLUGIN_API FMyViewExtension : public FViewExtension
{
public:
    FLensFlareSceneView(const FAutoRegister& AutoRegister);

    virtual void SubscribeToPostProcessingPass(EPostProcessingPass Pass, FAfterPassCallbackDelegateArray& InOutPassCallbacks, bool bIsPassEnabled) override;

protected:
    // Copied from PixelShaderUtils
    template <typename TShaderClass>
    static void AddFullscreenPass(
        FRDGBuilder& GraphBuilder,
        const FGlobalShaderMap* GlobalShaderMap,
        FRDGEventName&& PassName,
        const TShaderRef<TShaderClass>& PixelShader,
        typename TShaderClass::FParameters* Parameters,
        const FIntRect& Viewport,
        FRHIBlendState* BlendState = nullptr,
        FRHIRasterizerState* RasterizerState = nullptr,
        FRHIDepthStencilState* DepthStencilState = nullptr,
        uint32 StencilRef = 0
    );

    template <typename TShaderClass>
    static void DrawFullscreenPixelShader(
        FRHICommandList& RHICmdList,
        const FGlobalShaderMap* GlobalShaderMap,
        const TShaderRef<TShaderClass>& PixelShader,
        const typename TShaderClass::FParameters& Parameters,
        const FIntRect& Viewport,
        FRHIBlendState* BlendState = nullptr,
        FRHIRasterizerState* RasterizerState = nullptr,
        FRHIDepthStencilState* DepthStencilState = nullptr,
        uint32 StencilRef = 0
    );

    static inline void DrawFullScreenTriangle(FRHICommandList& RHICmdList, uint32 InstanceCount);

    // A delegate that is called when the Tone mapper pass finishes
    FScreenPassTexture TrianglePass_RenderThread(FRDGBuilder& GraphBuilder, const FSceneView& View, const FPostProcessMaterialInputs& Inputs);

    // For now we try to build a vertex buffer and see if it puts some shit in it
public:
    static void RenderTriangle(
        FRDGBuilder& GraphBuilder,
        const FGlobalShaderMap* ViewShaderMap,
        const FIntRect& View,
        const FScreenPassTexture& SceneColor
    );
};
```

Remember the order starts with adding a pass by giving the RDG lambda function the resources and parameters it needs. Next is defining the draw call(s) where
you setup the GPU Pipeline. 

You **MUST** setup the GPU pipeline because it defines all the parameters for the draw call (Blend States, Rasterizer State, Primitive type, Shaders, Viewport, Render Targets, Commands). 

The GPU Pipeline stores the state to be interpreted for the low lever graphics API calls being used.

First function is the templated `AddFullscreenPass` with the required parameters needed for the lambda function, nulling parameters we won’t use for the draw
call.

```cpp
// MyViewExtension.cpp
template <typename TShaderClass>
void FMyViewExtension::AddFullscreenPass(
    FRDGBuilder& GraphBuilder,
    const FGlobalShaderMap* GlobalShaderMap,
    FRDGEventName&& PassName,
    const TShaderRef<TShaderClass>& PixelShader,
    typename TShaderClass::FParameters* Parameters,
    const FIntRect& Viewport,
    FRHIBlendState* BlendState,
    FRHIRasterizerState* RasterizerState,
    FRHIDepthStencilState* DepthStencilState,
    uint32 StencilRef)
{
    check(PixelShader.IsValid());
    ClearUnusedGraphResources(PixelShader, Parameters);

    GraphBuilder.AddPass(
        Forward<FRDGEventName>(PassName),
        Parameters,
        ERDGPassFlags::Raster,
        [Parameters, GlobalShaderMap, PixelShader, Viewport, BlendState, RasterizerState, DepthStencilState, StencilRef]
        (FRHICommandList& RHICmdList)
        {
            FLensFlareSceneView::DrawFullscreenPixelShader<TShaderClass>(
                RHICmdList, GlobalShaderMap, PixelShader, *Parameters, Viewport,
                BlendState, RasterizerState, DepthStencilState, StencilRef);
        });
}
```


### Setup the Draw Call

```cpp
template <typename TShaderClass>
void FMyViewExtension::DrawFullscreenPixelShader(
    FRHICommandList& RHICmdList,
    const FGlobalShaderMap* GlobalShaderMap,
    const TShaderRef<TShaderClass>& PixelShader,
    const typename TShaderClass::FParameters& Parameters,
    const FIntRect& Viewport,
    FRHIBlendState* BlendState,
    FRHIRasterizerState* RasterizerState,
    FRHIDepthStencilState* DepthStencilState,
    uint32 StencilRef)
{
    check(PixelShader.IsValid());

    RHICmdList.SetViewport(
        (float)Viewport.Min.X, (float)Viewport.Min.Y, 0.0f,
        (float)Viewport.Max.X, (float)Viewport.Max.Y, 1.0f);

    // Begin Setup Gpu Pipeline for this Pass
    FGraphicsPipelineStateInitializer GraphicsPSOInit;
    TShaderMapRef<FTriangleVS> VertexShader(GlobalShaderMap);

    RHICmdList.ApplyCachedRenderTargets(GraphicsPSOInit);

    GraphicsPSOInit.BlendState = TStaticBlendState<>::GetRHI();
    GraphicsPSOInit.RasterizerState = TStaticRasterizerState<>::GetRHI();
    GraphicsPSOInit.DepthStencilState = TStaticDepthStencilState<false, CF_Always>::GetRHI();
    GraphicsPSOInit.BoundShaderState.VertexDeclarationRHI = GTriangleVertexDeclaration.VertexDeclarationRHI;
    GraphicsPSOInit.BoundShaderState.VertexShaderRHI = VertexShader.GetVertexShader();
    GraphicsPSOInit.BoundShaderState.PixelShaderRHI = PixelShader.GetPixelShader();
    GraphicsPSOInit.PrimitiveType = PT_TriangleList;

    GraphicsPSOInit.BlendState = BlendState ? BlendState : GraphicsPSOInit.BlendState;
    GraphicsPSOInit.RasterizerState = RasterizerState ? RasterizerState : GraphicsPSOInit.RasterizerState;
    GraphicsPSOInit.DepthStencilState = DepthStencilState ? DepthStencilState : GraphicsPSOInit.DepthStencilState;

    // End Gpu Pipeline setup
    SetGraphicsPipelineState(RHICmdList, GraphicsPSOInit, StencilRef);
    SetShaderParameters(RHICmdList, PixelShader, PixelShader.GetPixelShader(), Parameters);
    DrawFullScreenTriangle(RHICmdList, 1);
}

```

I created a `RenderTriangle` function to wrap my Add Pass and Draw Call Functions

```cpp
// FMyViewExtension.cpp
void FMyViewExtension::RenderTriangle(
    FRDGBuilder& GraphBuilder,
    const FGlobalShaderMap* ViewShaderMap,
    const FIntRect& ViewInfo,
    const FScreenPassTexture& SceneColor)
{
    // Begin Setup
    // Shader Parameter Setup
    FTrianglePSParams* PassParams = GraphBuilder.AllocParameters<FTrianglePSParams>();
    
    // Set the Render Target In this case is the Scene Color
    PassParams->RenderTargets[0] = FRenderTargetBinding(SceneColor.Texture, ERenderTargetLoadAction::ENoAction);

    // Create FTrianglePS Pixel Shader
    TShaderMapRef<FTrianglePS> PixelShader(ViewShaderMap);

    // Add Pass
    AddFullscreenPass<FTrianglePS>(GraphBuilder,
        ViewShaderMap,
        RDG_EVENT_NAME("TranglePass"),
        PixelShader,
        PassParams,
        ViewInfo);
}
```

### Binding the Delegate

I created the `TrianglePass_RenderThread` function.

```cpp
FScreenPassTexture FMyViewExtension::TrianglePass_RenderThread(FRDGBuilder& GraphBuilder, const FSceneView& View, const FPostProcessMaterialInputs& InOutInputs)
{
    const FScreenPassTexture SceneColor = InOutInputs.GetInput(EPostProcessMaterialInput::SceneColor);

    RDG_GPU_STAT_SCOPE(GraphBuilder, TrianglePass)
    RDG_EVENT_SCOPE(GraphBuilder, "TrianglePass");

    // Casting the FSceneView to FViewInfo
    const FIntRect ViewInfo = static_cast<const FViewInfo&>(View).ViewRect;
    const FGlobalShaderMap* ViewShaderMap = static_cast<const FViewInfo&>(View).ShaderMap;

    RenderTriangle(GraphBuilder, ViewShaderMap, ViewInfo, SceneColor);

    return SceneColor;
}
```

Before calling `RenderTriangle`, you need to get the `SceneColor` texture and perform a couple `static_casts`; one for the viewport dimensions and the other for a pointer to the global shader map. The global shader map points to our custom shader objects we declared using the global macros.

Lastly add the delegate function `TrianglePass_RenderThread` inside the if statement I talked about before.

```cpp
void FLensFlareSceneView::SubscribeToPostProcessingPass(EPostProcessingPass Pass, FAfterPassCallbackDelegateArray&
InOutPassCallbacks, bool bIsPassEnabled)
{
	if (Pass == EPostProcessingPass::Tonemap)
	{
		InOutPassCallbacks.Add(FAfterPassCallbackDelegate::CreateRaw(this, &FLensFlareSceneView::TrianglePass_RenderThread));
	}
}
```

**Compile**

You should see the triangle drawn to the viewport of your editor.

![[Unreal Engine Render Dependency Graph/Diagrams/TriangleOutput.png]](https://github.com/staticJPL/Render-Dependency-Graph-Documentation/blob/5110a92b8c25e1eab6f28e73456570649c2d0470/Diagrams/TriangleOutput.png)


