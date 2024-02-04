# Render Hardware Interface (RHI)

The original RHI was designed based on the D3D11 API, including some resource management and command interfaces. Since Unreal Engine is a ubiquitous
tool that supports many platforms like mobile, console and PC in which then can use (DirectX, Vulkan, OpenGL, Metal). To address this Unreal abstracted an
interface between all these API’s so they could keep the Rendering code as comprehensive as possible.

This achieved using different render threads as shown below:

![[Unreal Engine Render Dependency Graph/Diagrams/RHIUnrealDiagram.png]]

We have a Game thread, Render thread and RHI Thread. The important thing to understand is that anything that’s rendered has a twin object between game
thread and the render thread aside from some other specific cases.

**Game Thread**
- Primitive Components
- Light Components

**Render Thread**
- Primitive Proxy
- Light Proxy
  
**RHI Thread**
- Translates RHI “immediate” instructions from the Rendering thread to GPU based on the API specified. Note RHI Immediate really means immediate it’s not the same as a regular RHI command which is usually deferred.
- DX12, Vulkan and Host support parallelism and if a RHI immediate instruction is a generated parallel command then the RHI thread will translate it in parallel.

![[Unreal Engine Render Dependency Graph/Diagrams/Parallel CommandList RHI.png]]

## Basics of RHI

**FRenderResource**

The `FRenderResource` is a rendering resource representation on the rendering thread. This resource is managed and passed by the rendering thread as the
intermediate data between the game thread and

```cpp
/**
* A rendering resource which is owned by the rendering thread.
* NOTE - Adding new virtual methods to this class may require stubs added to FViewport/FDummyViewport, otherwise certain
modules may have link errors
*/
class RENDERCORE_API FRenderResource
{
public:
////////////////////////////////////////////////////////////////////////////////////
// The following methods may not be called while asynchronously initializing / releasing render resources.
/** Release all render resources that are currently initialized. */
static void ReleaseRHIForAllResources();
/** Initialize all resources initialized before the RHI was initialized. */
static void InitPreRHIResources();
/**
* Initializes the dynamic RHI resource and/or RHI render target used by this resource.
* Called when the resource is initialized, or when reseting all RHI resources.
* Resources that need to initialize after a D3D device reset must implement this function.
* This is only called by the rendering thread.
*/
virtual void InitDynamicRHI() {}
/**
* Releases the dynamic RHI resource and/or RHI render target resources used by this resource.
* Called when the resource is released, or when reseting all RHI resources.
* Resources that need to release before a D3D device reset must implement this function.
* This is only called by the rendering thread.
*/
virtual void ReleaseDynamicRHI() {}
/**
* Initializes the RHI resources used by this resource.
* Called when entering the state where both the resource and the RHI have been initialized.
* This is only called by the rendering thread.
*/
virtual void InitRHI() {}
/**
* Releases the RHI resources used by this resource.
* Called when leaving the state where both the resource and the RHI have been initialized.
* This is only called by the rendering thread.
*/
virtual void ReleaseRHI() {}
/**
* Initializes the resource.
* This is only called by the rendering thread.
*/
virtual void InitResource();
/**
* Prepares the resource for deletion.
* This is only called by the rendering thread.
*/
virtual void ReleaseResource();
/**
* If the resource's RHI resources have been initialized, then release and reinitialize it. Otherwise, do nothing.
* This is only called by the rendering thread.
*/
void UpdateRHI();
(...)
};
```

There are many subclasses that inherit from this class so that the rendering thread can transfer data and operations of the game thread to the RHI thread at
different levels of abstraction.

**FRHIResource**

`FRHIResource` is used for reference counting, delayed deletion, tracking, runtime data and marking. `FRHIResource` can be divided into state blocks, shader
bindings, shaders, pipeline states, buffers, textures, views, and other miscellaneous items. It should be noted that we can create platform specific types with this class. Check the source for `FRHIUniformBuffer`.

```cpp
/** The base type of RHI resources. */
class RHI_API FRHIResource
{
public:
	UE_DEPRECATED(5.0, "FRHIResource(bool) is deprecated, please use FRHIResource(ERHIResourceType)")
	FRHIResource(bool InbDoNotDeferDelete=false)
	: ResourceType(RRT_None)
	, bCommitted(true)
#if RHI_ENABLE_RESOURCE_INFO
	, bBeingTracked(false)
#endif
{
}
FRHIResource(ERHIResourceType InResourceType)
	: ResourceType(InResourceType)
	, bCommitted(true)
#if RHI_ENABLE_RESOURCE_INFO
	, bBeingTracked(false)
#endif
{
#if RHI_ENABLE_RESOURCE_INFO
	BeginTrackingResource(this);
#endif
}
virtual ~FRHIResource()
{
	check(IsEngineExitRequested() || CurrentlyDeleting == this);
	check(AtomicFlags.GetNumRefs(std::memory_order_relaxed) == 0); // this should not have any outstanding refs
	CurrentlyDeleting = nullptr;
	#if RHI_ENABLE_RESOURCE_INFO
	EndTrackingResource(this);
	#endif
}
	FORCEINLINE_DEBUGGABLE uint32 AddRef() const
	{...};
private:
	// Separate function to avoid force inlining this everywhere. Helps both for code size and performance.
	inline void Destroy() const
	{...};
public:
	FORCEINLINE_DEBUGGABLE uint32 Release() const
	{...};
	FORCEINLINE_DEBUGGABLE uint32 GetRefCount() const
	{...};
	static int32 FlushPendingDeletes(FRHICommandListImmediate& RHICmdList);
	static bool Bypass();
	bool IsValid() const
	{...};
	void Delete()
	{...};
	inline ERHIResourceType GetType() const { return ResourceType; }
	#if RHI_ENABLE_RESOURCE_INFO
	// Get resource info if available.
	// Should return true if the ResourceInfo was filled with data.
	virtual bool GetResourceInfo(FRHIResourceInfo& OutResourceInfo) const
	{...};
	static void BeginTrackingResource(FRHIResource* InResource);
	static void EndTrackingResource(FRHIResource* InResource);
	static void StartTrackingAllResources();
	static void StopTrackingAllResources();
#endif
private:
	class FAtomicFlags
	{
	static constexpr uint32 MarkedForDeleteBit = 1 << 30;
	static constexpr uint32 DeletingBit = 1 << 31;
	static constexpr uint32 NumRefsMask = ~(MarkedForDeleteBit | DeletingBit);
	std::atomic_uint Packed = { 0 };
	public:
	int32 AddRef(std::memory_order MemoryOrder)
	{...};
	int32 Release(std::memory_order MemoryOrder)
	{...};
	bool MarkForDelete(std::memory_order MemoryOrder)
	{...};
	bool UnmarkForDelete(std::memory_order MemoryOrder)
	{...};
	bool Deleteing()
	{...};
	mutable FAtomicFlags AtomicFlags;
	const ERHIResourceType ResourceType;
	uint8 bCommitted : 1;
	#if RHI_ENABLE_RESOURCE_INFO
	uint8 bBeingTracked : 1;
	
#endif
	static std::atomic<TClosableMpscQueue<FRHIResource*>*> PendingDeletes;
	static FHazardPointerCollection PendingDeletesHPC;
	static FRHIResource* CurrentlyDeleting;
// Some APIs don't do internal reference counting, so we have to wait an extra couple of frames before deleting resources
// to ensure the GPU has completely finished with them. This avoids expensive fences, etc.
	struct ResourcesToDelete
	{...};
};
```

**FRHICommandList**

The RHI Command list is an instruction queue that is used to manage and execute a group of command objects. The parent class for this is
FRHICommandListBase. `FRHICommandListBase` defines the basic data (Command list, Device context) and interface (Command refresh, wait, enqueue,
memory allocation etc..) which is required by the command queue. FRHIComputeCommandList defines the interfaces between compute shaders, state transition of GPU resources and the settings for the shader parameters. FRHICommandList defines the interface of common rendering pipelines, these include binding `Vertex Shaders`, `Pixel Shaders`,` Geometry Shaders`, Primitive Drawing, shader parameters, resource management etc.

**RHIContext & DynamicRHI**

Lastly the `RHIContext` and `DynamicRHI` is another interface class which defines a set of graphics API related operations. As mentioned earlier some APIs can
process commands in parallel and so this separate object is used to define that.

In summary, the RHI classes are the lowest level abstraction used in Unreal to talk to our Graphics APIs. The ones listed are the main ones you should be aware
of. I’ve provided an in depth article that goes through the RHI in more detail under the references section. Also if the command lists doesn’t make sense just
look up how a basic command list/buffer is processed to the GPU. You can look at one of the Graphics API’s to see how it’s queued.

