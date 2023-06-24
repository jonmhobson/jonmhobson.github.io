---
layout: post
title: Raytracing in Metal Part 3 - Spheres!
---

This is part 3 of the Metal Raytracing series.

Most raytracing tutorials start off by raytracing the humble sphere, since it's mathematically the simplest of all shapes to define and intersect with. With Metal, if you want to test against a sphere you're going to need a custom intersector, but now that we have those under our belt it's time to raytrace the humble sphere!

## Spheres

We can define a sphere using only 4 values: The X, Y, and Z coordinates at its center, and its radius.

The formula for line-sphere intersection is relatively simple. If you're interested in how it works, you can read up at [https://en.wikipedia.org/wiki/Line–sphere_intersection]() but since this tutorial is about Metal all you need to know is that we are going to build a quadratic formula where:

*direction = the ray direction*

*oc = the vector from the center of the sphere to the ray origin*

*r = the sphere's radius*

* a = dot(direction, direction)
* b = 2 * dot(oc, direction)
* c = dot(oc, oc) - r^2

We then solve this equation as

* d = b * b - 4 * a * c

If d < 0 there is no intersection. Otherwise our intersection occured at (-b - sqrt(d)) / (2 * a). Simple!

## Shaders.metal

First lets add two new types at the top of our Shaders file

```
struct BoundingBoxIntersection {
    bool accept [[accept_intersection]];
    float distance [[distance]];
};

typedef struct Sphere {
    packed_float3 center;
    float radius;
} Sphere;
```

The first one is what we will return from our intersection function. As you can see from the annotations, *accept* tells Metal if our ray hit or not, and *distance* will tell Metal where along the ray the hit happened.

The second struct should be fairly self-evident, and is our Sphere structure.

Finally delete the old customIntersectionFunction and replace it with:

```
[[intersection(bounding_box)]]
BoundingBoxIntersection sphereIntersectionFunction(uint id [[primitive_id]],
                                                   float3 origin [[origin]],
                                                   float3 direction [[direction]],
                                                   float minDistance [[min_distance]],
                                                   float maxDistance [[max_distance]],
                                                   device Sphere* sphereBuffer [[buffer(0)]]) {
    BoundingBoxIntersection ret;

    float3 oc = origin - sphereBuffer[id].center;

    float a = dot(direction, direction);
    float b = 2 * dot(oc, direction);
    float c = dot(oc, oc) - sphereBuffer[id].radius * sphereBuffer[id].radius;

    float disc = b * b - 4 * a * c;

    if (disc <= 0.0) {
        ret.accept = false;
    } else {
        ret.distance = (-b - sqrt(disc)) / (2 * a);
        ret.accept = ret.distance >= minDistance && ret.distance <= maxDistance;
    }

    return ret;
}
```

The function is annotated as [[intersection(bounding_box)]] which tells Metal that this function should be called on a bounding box type. Metal's acceleration structures work with bounding boxes, not spheres, so later on we will add a bounding box to our acceleration structure that will represent our sphere. If the ray intersects that bounding box, then Metal will call this function and then we can work out if we hit our sphere or not.

Note the annotated parameters to this function. We are going to support multiple Spheres, so id [[primitive_id]] will tell us which one to test against. Origin and direction are for the ray we're going to test against, and minDistance and maxDistance allow Metal to set a bound so that if minDistance is say 0.2 and our intersection occurs at 0.1, we will say that the intersection didn't happen.

Finally we have our sphereBuffer, which will be an array of Sphere types that we will pass in through buffer(0).

## RaytracingRenderPass.swift

First off, lets load the correct intersection function. Replace 
```
let intersectionFunction = metal.library.makeFunction(name: "customIntersectionFunction")!
``` 
with
```
let intersectionFunction = metal.library.makeFunction(name: "sphereIntersectionFunction")!
```

Next let's create our Sphere struct. A common technique is to add C header file and then a bridging header so you can share code, but I'm not a huge fan of this since you end up with slightly annoying Swift code, also it's not the point of this series so I'm just going to duplicate it for now.

We're also going to need a bounding box, and Metal expects two packed float 3s. So let's add this to the top of the file.

```
struct BoundingBox {
    let min: MTLPackedFloat3
    let max: MTLPackedFloat3
}

struct Sphere {
    let center: MTLPackedFloat3
    let radius: Float

    var boundingBox: BoundingBox {
        .init(min: MTLPackedFloat3Make(center.x - radius, center.y - radius, center.z - radius),
              max: MTLPackedFloat3Make(center.x + radius, center.y + radius, center.z + radius))
    }
}
```

Thus we have a Sphere class that matches the one we have in our Shader, which also has a computed variable that will give us a bounding box by adding / subtracing the radius to the center to create a min and max.

Next let's create a small array of Spheres that we can put in a MTLBuffer and send in to our shaders. Add this under the previous code:

```
private let spheres: [Sphere] = [
    .init(center: MTLPackedFloat3Make(0.0, -100.5, -1.0), radius: 100.0),
    .init(center: MTLPackedFloat3Make(0.0, 0.0, -1.0), radius: 0.5)
]
```
This adds one huge sphere that will act as our "ground" and one small sphere sitting on top of our ground sphere just where our camera is looking.

Now we're going to put our Sphere data into a buffer, so add the following to the `RaytracingRenderPass` class

```
let sphereBuffer: MTLBuffer
```

And in the init function, add this line of code that will copy the sphere data into a brand new shared buffer.

```
sphereBuffer = metal.device.makeBuffer(bytes: spheres, length: MemoryLayout<Sphere>.stride * spheres.count)!
```

Next, we need to make this buffer available to our intersection function (which if you remember we want to put in as [[buffer(0)]], so just after we call renderPipeline.makeIntersectionFunctionTable:

```	
self.intersectionFunctionTable.setBuffer(sphereBuffer, offset: 0, index: 0)
```

Finally, we're going to need to add our bounding boxes to our acceleration structure, so lets pass our Sphere array in to the constructor of our SimplePrimitiveAccelerationStructure like so:

```
self.simplePrimitiveAccelStruct = SimplePrimitiveAccelerationStructure(metal: metal, spheres: spheres)
```

## SimplePrimitiveAccelerationStructure.swift

Let's change our init function to accept the sphere array we just passed in

```
init(metal: MetalDevice, spheres: [Sphere]) {
```

Then let's add our bounding boxes into an MTLBuffer

```
let boundingBoxes = spheres.map(\.boundingBox)

let boundingBoxBuffer = metal.device.makeBuffer(
    bytes: boundingBoxes,
    length: MemoryLayout<BoundingBox>.stride * spheres.count
)
```

Finally change our geometry descriptor from an `MTLAccelerationStructureTriangleGeometryDescriptor` into an `MTLAccelerationStructureBoundingBoxGeometryDescriptor` as follows:

```
let geometryDescriptor = MTLAccelerationStructureBoundingBoxGeometryDescriptor()
geometryDescriptor.boundingBoxBuffer = boundingBoxBuffer
geometryDescriptor.boundingBoxCount = boundingBoxes.count
geometryDescriptor.intersectionFunctionTableOffset = 0
```

Delete all the old code creating the vertexBuffer and also setting .triangleCount and .opaque as these are no longer needed.

## Shaders.metal

The only thing remaining now is to change our fragmentShader, as it's no longer expecting an intersection_type::triangle, but an intersection_type::bounding_box instead.

Also we no longer have barycentric coordinates, so lets change our shader to the following:

```
fragment float4 fragmentShader(VertexOut in [[stage_in]],
                               acceleration_structure<> accelStructure [[buffer(0)]],
                               intersection_function_table<triangle_data> intersectionFunctionTable [[buffer(1)]]) {

    ray r;

    r.origin = float3(0, 0, 3);
    r.direction = float3(in.uv, -1);

    intersector<triangle_data> intersector;
    intersection_result<triangle_data> intersection;

    intersection = intersector.intersect(r, accelStructure, intersectionFunctionTable);

    if (intersection.type == intersection_type::bounding_box) {
        return float4(1);
    } else {
        return float4(0);
    }
}
```

Now if we get a hit, we're returning white, and if we don't, we're returning black.

This give us:

![](https://github.com/jonmhobson/jonmhobson.github.io/blob/master/images/spheres.png?raw=true)

Which is exactly what we were expecting! A small sphere sitting on top of a huge ground sphere.

It's not very interesting though, so lets add a nice background by replacing the miss path with the following code as an homage to Raytracing in one week.

```
float3 unitDir = normalize(r.direction);
float t = 0.5 * (unitDir.y + 1.0);
float3 col = (1.0 - t) * float3(1.0, 1.0, 1.0) + t * float3(0.5, 0.7, 1.0);
return float4(col, 1.0);
```

![](https://github.com/jonmhobson/jonmhobson.github.io/blob/master/images/sphereSky.png?raw=true)

Much better. But before we can make our hits look as good as our misses, we're going to need access to our Sphere array.

In the `encode` function of our `RaytracingRenderPass` class add the following line:

```
renderEncoder.setFragmentBuffer(sphereBuffer, offset: 0, index: 2)
```

This will pass in our sphere buffer into [[buffer(2)]], so let's add it as a parameter to our fragmentShader's signature

```
fragment float4 fragmentShader(VertexOut in [[stage_in]],
                               acceleration_structure<> accelStructure [[buffer(0)]],
                               intersection_function_table<triangle_data> intersectionFunctionTable [[buffer(1)]],
                               device Sphere * sphereBuffer [[buffer(2)]]) {
```

and add the following code to our fragment shader in the path where we actually did get a hit:

```
Sphere sphere = sphereBuffer[intersection.primitive_id];

float3 intersectionPoint = (r.origin + r.direction * intersection.distance);
float3 normal = normalize((intersectionPoint - sphere.center) / sphere.radius);
float3 color = 0.5 * (normal + float3(1));

return float4(color, 1);
```

We first get the intersection.primitive_id, which will tell us which of our two spheres the ray hit. We can then use this to get our sphere from the sphereBuffer.

Next we get the point at which our ray intersected our sphere, and we do this by feeding intersection.distance into the parametric form of our ray.

Finally we use the sneaky fact that the surface normal of a sphere is identical to the vector from its center to its surface. We divide by the radius to be good, but since we're normalizing it it's not really necessary.

Finally we transform our normal into a color (a normal can go from -1 to 1, so we add 1 and then divide by 2 to get it in the range 0...1) and then return the normal as a color.

This gives us the following masterpiece, which will look familiar to anyone who has worked their way through Raytracing in One Weekend.

![](https://github.com/jonmhobson/jonmhobson.github.io/blob/master/images/sphereNormals.png?raw=true)
