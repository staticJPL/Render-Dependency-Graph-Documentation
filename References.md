# References

In order for me piece this all together it took a bit of time digging through Engine Source code and googling articles. In this section I will link the references I
bookmarked and put them in the order of learning. This way you can review this document in order of the references which build upon the knowledge you need
to understand rendering in Unreal and basics for Graphic Programming.

### Basic Graphics Programming References
--------------------------------------------------
**DirectX Pipeline**
https://www.youtube.com/watchv=pfbWt1BnPIo&list=PLqCJpWy5Fohd3S7ICFXwUomYW0Wv67pDD&index=21&ab_channel=ChiliTomatoNoodle

**DirectX Architecture/Swap Chain**
https://www.youtube.com/watch?v=bMxNN9dO4cI&list=PLqCJpWy5Fohd3S7ICFXwUomYW0Wv67pDD&index=19&ab_channel=ChiliTomatoNoodle

**Vulkan Uniform Buffers**
https://www.youtube.com/watch?v=may_GMkfs5k&t=607s&ab_channel=BrendanGalea

**Vulkan Index and Staging Buffers**
https://www.youtube.com/watch?v=qxuvQVtehII&ab_channel=BrendanGalea

**Deferred Rendering**
https://www.youtube.com/watch?v=n5OiqJP2f7w&ab_channel=BenAndrew

Shader Basics
---------------------------------------------------------------------------
https://www.youtube.com/watch?v=kfM-yu0iQBk&ab_channel=FreyaHolm%C3%A9r

HLSL Semantics
https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics

Unreal Engine Passes & Rendering Basics
---------------------------------------------------------------------

Unreal Engines Render Passes
https://unrealartoptimization.github.io/book/profiling/passes/

Unreal Engine Rendering (Drawing Policies are Deprecated)
https://medium.com/@lordned/unreal-engine-4-rendering-overview-part-1-c47f2da65346

Unreal Engine Vertex Factories
(Basically how you’re suppose to create vertex buffers, there are macros for this which you pass to RDG)

https://medium.com/realities-io/creating-a-custom-mesh-component-in-ue4-part-1-an-in-depth-explanation-of-vertex-factories-4a6fd9fd58f2

Unreal Engine RHI
----------------------------------------------
Analysis of Unreal Engine Rendering System: (Chinese) timlly-chang
https://www.cnblogs.com/timlly/p/15156626.html#101-%E6%9C%AC%E7%AF%87%E6%A6%82%E8%BF%B0

RDG Architecture by Riccardo Loggini
https://logins.github.io/graphics/2021/05/31/RenderGraphs.html

RDG Analysis (Chinese)
https://blog.csdn.net/qjh5606/article/details/118246059
https://juejin.cn/post/7085216072202190856

Shaders with RDG
https://logins.github.io/graphics/2021/03/31/UE4ShadersIntroduction.html

Render Architecture
https://ikrima.dev/ue4guide/graphics-development/render-architecture/base-usf-shaders-code-flow/

Epic Games RDG Crash Course
https://epicgames.ent.box.com/s/ul1h44ozs0t2850ug0hrohlzm53kxwrz

Epic Games RDG Documentation
https://docs.unrealengine.com/5.1/en-US/render-dependency-graph-in-unreal-engine/

Scene View Extension Class
------------------------------------------
Forum Post
https://forums.unrealengine.com/t/using-sceneviewextension-to-extend-the-rendering-system/600098

Caius’ Blog Global Shaders using Scene View Extension Class
https://itscai.us/blog/post/ue-view-extensions/
