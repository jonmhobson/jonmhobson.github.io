---
layout: post
title: Raytracing in Metal Part 1 - Hello Triangle
---

In part 0 we built a very simple Swift application that uses Metal to render a simple quad (two triangles) that covers the entire screen.

This causes our fragment shader to be called for every single on-screen pixel, and we're going to use this to do some raytracing!

First things first, we're going to need to build an "Acceleration Structure". Acceleration structures can be a complex beast with a whole host of confusing configurations, but for now we're just going to build the simplest acceleration structure we can think of - a Primitive acceleration structure containing a single triangle.

Ok, but what is an acceleration structure? It's essentially just a heirarchical scene graph that you give to Metal. Metal will then take your data and opaquely order it in a way such that you can perform ray intersection tests with this scene as fast as possible (hence the name *acceleration* structure)

If you have previously done ray tracing, you might have played around at creating a BVH or Bounding Volume Heirarchy, so that when you want to test against N triangles, you don't need to linearly loop through N triangles and test every single one against your ray in an O(n) fashion.

Instead you would create a heirarchical tree structure so that if you know that your ray does not intersect with the parent bounding box, you know that the ray will not intersect with its children either. Turning the algorithm from an O(n) slog into a much faster O(log n) affair.

Fortunately Metal handles all this for us, and essentially all we need to do is pass in our geometry and let Metal work its magic.

Lets start by making a class to encapsulate our simple acceleration structure.

```
import Metal

class SimplePrimitiveAccelerationStructure {
    private let metal: MetalDevice

    private let vertices: [MTLPackedFloat3] = [
        MTLPackedFloat3Make( 0.0,  1.0, 0.0),
        MTLPackedFloat3Make(-1.0, -1.0, 0.0),
        MTLPackedFloat3Make( 1.0, -1.0, 0.0)
    ]

    init(metal: MetalDevice) {
        self.metal = metal
    }

    func build() {
 
    }
}

```

Right now we're passing in our metal device objects, and we have a simple array of 3 vertices representing our triangle that we'll put into a buffer to pass to the acceleration structure.

We also have a build function that will actually build the acceleration structure. Building an acceleration structure is a potentially time-consuming operation that is done on the GPU so we want maximum control over when this is called.

First things first we need to put our vertices into a MTLBuffer. In init, just after `self.metal = metal` we add

```
let vertexBuffer = metal.device.makeBuffer(
    bytes: vertices,
    length: MemoryLayout<MTLPackedFloat3>.stride * vertices.count
)!
```

This will copy our three vertices into a shared (accessible by both CPU and GPU) MTLBuffer. We're using MTLPackedFloat3 because most three element vectors add a hidden 4th component to maintain 16 byte alignment. However we literally just want our array to be 3x3 floats.

Metal does allow you to specify different vertex formats should you want to pass in say Float16 / half precision values, but for now we'll just use the default which is MTLPackedFloat3.

## Geometry Descriptor

Next up we will create our geometry descriptor.

```
let geometryDescriptor = MTLAccelerationStructureTriangleGeometryDescriptor()
geometryDescriptor.vertexBuffer = vertexBuffer
geometryDescriptor.triangleCount = 1
```

The interesting thing here is the type **MTLAccelerationStructureTriangleGeometryDescriptor**. This lets metal know that we're sending in Triangle geometry.

The rest is as simple as it looks. We're literally just passing in our single-triangle vertex buffer and telling metal that there is 1 triangle inside it.

This is also where you could specify the format of our vertex buffer if you wanted to.
```
geometryDescriptor.vertexFormat = .float3
```
Feel free to play around with this. The cool thing about Swift is that it supports Float16 by default, so you could for example switch out our vertices with:

```
private let vertices: [Float16] = [
     0.0,  1.0, 0.0,
    -1.0, -1.0, 0.0,
    1.0, -1.0, 0.0
]

```
and set the vertex format to `geometryDescriptor.vertexFormat = .half3` and it should work identically.

## Allocating the acceleration structure

Add the following member variables to our `SimplePrimitiveAccelerationStructure` class.

```
let accelerationStructure: MTLAccelerationStructure

private let accelerationStructureDescriptor = MTLPrimitiveAccelerationStructureDescriptor()
private let accelStructSizes: MTLAccelerationStructureSizes
```

It's important to note that what we're doing now is "priming" the acceleration structure to be built. Even though we're going to call a function called "makeAccelerationStructure" the acceleration structure itself won't be usable until we add the code to our build function.

Add the remaining code to the bottom of our `init` function

```
self.accelerationStructureDescriptor.geometryDescriptors = [geometryDescriptor]
self.accelStructSizes = metal.device.accelerationStructureSizes(descriptor: accelerationStructureDescriptor)
self.accelerationStructure = metal.device.makeAccelerationStructure(size: accelStructSizes.accelerationStructureSize)!
```

The first line sets up our accelerationStructureDescriptor. This is an object of type MTLPrimitiveAccelerationStructureDescriptor - the important keyword here is Primitive, and it tells Metal that this acceleration structure is going to contain geometry. Later on we'll see Instance type acceleration structures, which can contain other acceleration structures, but for now we're sticking to the Primitive type.

We pass in the geometry descriptor that we created in the previous step. Note that you can pass in multiple geometry types, but for now we only need one.

The second line creates an MTLAccelerationStructureSizes object. We are asking Metal: "Hey Metal, how much memory do you need to build an acceleration structure containing this geometry?". Metal will then let us know how much memory it will need for the acceleration structure itself, and also how big of a scratch buffer we will need to send in to the GPU in order to build it.

There is also the refitScratchBufferSize, but we'll ignore that for now since we won't be refitting this structure.

Finally the third line allocates the acceleration structure itself. All ready for building.

## Building the acceleration structure

Ok, now we've gotten to the exciting part where we can actually ask Metal to build the acceleration structure for our geometry.

```
func build() {
    let scratchBuffer = metal.device.makeBuffer(length: accelStructSizes.buildScratchBufferSize, options: .storageModePrivate)!
    let commandBuffer = metal.commandQueue.makeCommandBuffer()!
    let commandEncoder = commandBuffer.makeAccelerationStructureCommandEncoder()!

    commandEncoder.build(accelerationStructure: accelerationStructure,
                         descriptor: accelerationStructureDescriptor,
                         scratchBuffer: scratchBuffer,
                         scratchBufferOffset: 0)

    commandEncoder.endEncoding()
    commandBuffer.commit()
}
```

First we create a scratchBuffer. We can make this private because it's only going to be used by the GPU and we never need to access it using the CPU.

Next we create a stand-alone commandBuffer, because we want to just send this job off to the GPU one-and-done. Also we could call ```commandBuffer.waitUntilCompleted()``` if we wanted to wait synchronously until the acceleration structure is definitely built.

Then we make an "acceleration structure command encoder" which is a special type of command encoder that essentially just lets us send acceleration structure specific commands to the GPU. You may remember from part 0 that we created a renderCommandEncoder for performing render commands, well this is the acceleration structure version of that.

We then use our commandEncoder to actually build the acceleration structure.

Finally we close our encoder with endEncoding like good Metal citizens, and finally `commandBuffer.commit()` to send it off to the GPU.

We should now (well, technically after the GPU has asynchronously completed) have a complete primitive acceleration structure containing a single triangle that is ready to be passed into our shader!

## Modifying RaytracingRenderPass

To actually use our newly created `SimplePrimitiveAccelerationStructure` we must first add it as a member of our RaytracingRenderPass class

```
let simplePrimitiveAccelStruct: SimplePrimitiveAccelerationStructure
```

At the end of our `init` function we add the following code

```
self.simplePrimitiveAccelStruct = SimplePrimitiveAccelerationStructure(metal: metal)
self.simplePrimitiveAccelStruct.build()
```

This allocates our SimplePrimitiveAccelerationStructure and then builds it.

Finally in the `encode` function we add a call to `renderEncoder.setFragmentAccelerationStructure` where we pass in our shiny new acceleration structure so that it will be available to us in the fragment shader in bufferIndex 0.

```
func encode(into renderEncoder: MTLRenderCommandEncoder) {
    renderEncoder.setRenderPipelineState(renderPipeline)
    renderEncoder.setFragmentAccelerationStructure(simplePrimitiveAccelStruct.accelerationStructure, bufferIndex: 0)
    renderEncoder.drawPrimitives(type: .triangle, vertexStart: 0, vertexCount: 6)
}
```

## Fragment Shader

Firstly let metal know you want to use raytracing functions in your shader by adding this at the top of your shaders.metal file.

```
using namespace raytracing;
```

Secondly, we have to receive the acceleration structure that we just put into buffer slot 0. This means changing the function signature to look like this:

```
fragment 
float4 fragmentShader(VertexOut in [[stage_in]],
                      acceleration_structure<> accelStructure [[buffer(0)]])
```

The annotation [[buffer(0)]] tells Metal  that the acceleration structure was placed into buffer slot 0. It's technically unnecessary, because if you omit it Metal will just guess that 0 is going to be the first parameter, 1 will be the second, and so on. But it's very good practice to actually annotate which parameter maps to which buffer so your code is a lot clearer.

The type `acceleration_structure<>` tells Metal that this is the simplest possible acceleration structure. The acceleration_structure type accepts various tags as template parameters. The metal_raytracing header contains a type alias `using primitive_acceleration_structure = acceleration_structure<>;` so you could use `primitive_acceleration_structure` instead of `acceleration_structure<>` to be even more explicit.

## Raytracing!

Ok, now we finally have everything we need in our fragmentShader to do some raytracing!

First off we're going to need a ray.

```
ray r;

r.origin = float3(0, 0, 3);
r.direction = float3(in.uv, -1);
```

If you remember back to part 0 where we discussed the vertexShader, you'll remember that in.uv represents the position of the full-screen quad we're rendering to the entire screen. i.e. it goes from -1, -1 in the bottom left, all the way to 1, 1 in the top right.

Therefore we're building a ray that originates at (0, 0, 3), sitting 3 units high in the Z axis, with a direction going from (-1, -1, -1) to (1, 1, -1).

Our triangle is sitting at Z = 0, 2 units wide in the X axis and 2 units high in the Y.

![https://github.com/jonmhobson/jonmhobson.github.io/blob/0dd7bf095a2f503cd10cdb451f5fbfddf94cd8bc/images/Frustum.jpg?raw=true](https://github.com/jonmhobson/jonmhobson.github.io/blob/0dd7bf095a2f503cd10cdb451f5fbfddf94cd8bc/images/Frustum.jpg?raw=true)

Now we're going to need an intersector and an intersector result

```
intersector<triangle_data> intersector;
intersection_result<triangle_data> intersection;
```

We're passing in the triangle_data tag, to let Metal know that we're looking for triangles.

Finally

```
intersection = intersector.intersect(r, accelStructure);
```

Bam! Every single pixel in our image is sending its own ray through our acceleration structure and returning an intersection_result.

Now the only thing to do is to interpret that result.

```
if (intersection.type == intersection_type::triangle) {
    return float4(intersection.triangle_barycentric_coord, 1, 1);
} else {
    return float4(in.uv, 0, 0);
}
```

If the ray hit our triangle, we return a colour from the triangle's barycentric coordinates. Otherwise we just return a color from our in UVs.

Barycentric coordinates sound scary, but they literally just mean the weights of each vertex. The x value means how close we are to vertex 1, the y value means how close we are to vertex 2, and if you want to know how close we are to vertex 0, you just subtract the other two from 1.

For example:

* (0, 0) = vertex 0
* (1, 0) = vertex 1
* (0, 1) = vertex 2

You can probably see how you can use these two values to represent any position on the triangle.

![https://github.com/jonmhobson/jonmhobson.github.io/blob/master/images/HelloTriangle.jpg?raw=true](https://github.com/jonmhobson/jonmhobson.github.io/blob/master/images/HelloTriangle.jpg?raw=true)

Sweet! It may not look like much, but this is a huge breakthrough! If you can raytrace one triangle you can raytrace millions, and this gives us a really strong foundation to build on.
