---
layout: post
title: Raytracing in Metal Part 2 - Custom intersectors
---

In part 2 of this Raytracing in Metal series we're going to do a little bit of a detour and work on writing a custom intersector function. It's not going to be that useful at this point, but it's a good thing to learn right now when the app is still very simple.

## What is a custom intersector?

Right now, when we fire a ray at our triangle there are two possible outcomes: either we hit the triangle, or we didn't.

That's all very well and good when we're rendering opaque meshes, but what if we want to render a leaf? In a normal raster engine we'd likely just put a leaf texture onto some simple polygon geometry and then use the alpha channel to refrain from drawing pixels where the alpha value is 0 - allowing us to see things that are behind our leaf.

Unfortunately things are a bit more complicated with raytracing, because the shading part comes AFTER we've done the intersection. This means that even if we don't render the transparent parts of the texture, our rays are never going to hit the geometry behind.

What we need is some kind of function we can pass into Metal that would allow Metal to ask US if the ray truly did hit this particular triangle.

Fortunately that's exactly what a custom intersector is!

We're starting off exactly where we left off in part 1.

## Shaders.metal

First off let's make the absolute simplest intersection function we can. This is simply a function that always returns false. We'll know that this function is being called when our triangle disappears!

```
[[intersection(triangle, triangle_data)]]
bool customIntersectionFunction(float2 barycentricCoords [[barycentric_coord]]) {
    return false;
}
```

The annotation [[intersection(triangle, triangle_data)]] tells Metal that we're expecting triangles, and the triangle_data tells Metal that if it could generate us some barycentric coordinates and pop them into the parameter that we've annotated with [[barycentric_coord]] then that would be lovely.

We'll use the barycentric coordinates later, but for now we're happy just passing them in.

## RaytracingRenderPass.swift

Now we need to actually somehow link our "customIntersectionFunction" so Metal can find it. You may remember we loaded our vertex and fragment shaders in the initializer of RaytracingRenderPass, and then passed them into the RenderPipelineState. So that's where we'll handle our new function.

Unfortunately things aren't quite as simple as calling renderDescriptor.intersectionFunction, but it's not too bad. The following code goes after `renderDescriptor.colorAttachments[0].pixelFormat = pixelFormat` in our initializer.

```
let intersectionFunction = metal.library.makeFunction(name: "customIntersectionFunction")!

let linkedFunctions = MTLLinkedFunctions()
linkedFunctions.functions = [intersectionFunction]

renderDescriptor.fragmentLinkedFunctions = linkedFunctions
```

Here we are loading the customIntersectionFunction, adding it to the functions array of an MTLLinkedFunctions object, and then setting it as a fragmentLinkedFunction on our renderDescriptor.

Next, we need to build something called an intersection function table. Currently we only need 1 intersection function, but you can imagine that in a production renderer you might have tens or even hundreds of different intersection functions.

First add the following member to our RaytracingRenderPass class
```
let intersectionFunctionTable: MTLIntersectionFunctionTable
```

Add the following code to the end of our initializer:

```
let intersectionFunctionTableDescriptor = MTLIntersectionFunctionTableDescriptor()
intersectionFunctionTableDescriptor.functionCount = 1
self.intersectionFunctionTable = renderPipeline.makeIntersectionFunctionTable(descriptor: intersectionFunctionTableDescriptor, stage: .fragment)!

let functionHandle = renderPipeline.functionHandle(function: intersectionFunction, stage: .fragment)
intersectionFunctionTable.setFunction(functionHandle, index: 0)
```

The first part creates a descriptor and tells it that we have 1 function. We then use the renderPipeline to actually allocate the intersection function table. We tell Metal that we're going to use this function in the fragment stage.

Next we get a handle to the function that we linked to our RenderPipeline. If we hadn't linked this function correctly, then attempting to get the functionHandle would return nil.

We then bind this function to index 0 in the function table. When we actually want to tell Metal "Hey, use our customIntersectionFunction with this geometry!" we will just use say "Hey, use function 0"

Finally in the encode function we must add the following line just after we call `setFragmentAccelerationStructure`

```
renderEncoder.setFragmentIntersectionFunctionTable(intersectionFunctionTable, bufferIndex: 1)
```

And make sure that we receive this intersection function table in our fragmentShader by changing the function signature to 

```
fragment float4 fragmentShader(VertexOut in [[stage_in]],
                               acceleration_structure<> accelStructure [[buffer(0)]],
                               intersection_function_table<triangle_data> intersectionFunctionTable [[buffer(1)]])
```

## SimplePrimitiveAccelerationStructure.swift

Now the only thing left to do is that when we put our triangle data into our acceleration structure using our **MTLAccelerationStructureTriangleGeometryDescriptor**, we must also tell it that we want to use a customer intersection function, and that we want to use function index 0.

Therefore we change our geometryDescriptor to look like this:

```
geometryDescriptor.vertexBuffer = vertexBuffer
geometryDescriptor.triangleCount = 1
geometryDescriptor.opaque = false
geometryDescriptor.intersectionFunctionTableOffset = 0
```

The last two lines are new. Firstly, we tell Metal that this geometry is not opaque. If you mark the geometry as opaque then Metal will skip the intersection function even if you try to use one. We then tell metal that we would like to use the function at index 0, which we set in the previous section.

## Back to Shaders.metal

Finally we're ready to use our intersection function, which is actually incredibly easy.

In the fragmentShader function, simply pass in the intersectionFunctionTable as the third parameter of our intersect function call.

From:

```
intersection = intersector.intersect(r, accelStructure);
```

To:

```
intersection = intersector.intersect(r, accelStructure, intersectionFunctionTable);
```

Now, if everything has gone to plan, your triangle should have disappeared!

![https://github.com/jonmhobson/jonmhobson.github.io/blob/master/images/raytracing2_0.png?raw=true](https://github.com/jonmhobson/jonmhobson.github.io/blob/master/images/raytracing2_0.png?raw=true)

However, we want to be completely sure that the function is working so let's try the following:

```
[[intersection(triangle, triangle_data)]]
bool customIntersectionFunction(float2 barycentricCoords [[barycentric_coord]]) {
    float3 barycentric = float3(1.0 - barycentricCoords.x - barycentricCoords.y, barycentricCoords);
    float dist = length(float3(0.5) - barycentric);
    return dist > 0.5;
}
```

![https://github.com/jonmhobson/jonmhobson.github.io/blob/master/images/raytracing2_1.png?raw=true](https://github.com/jonmhobson/jonmhobson.github.io/blob/master/images/raytracing2_1.png?raw=true)

This function unpacks our barycentric coordinates into a float3, and then compares the distance to (0.5, 0.5, 0.5). If the distance is greater than 0.5 we say we hit, if not then we say we missed. This cuts a nice circular hole in the middle of our triangle and proves that our intersector is working as expected.

Eventually you'll be able to use this for alpha stencilling, but texture-mapping will have to come later.
