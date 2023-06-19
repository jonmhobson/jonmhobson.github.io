---
layout: post
title: Raytracing in Metal Part 0 Intro
---

# Raytracing in Metal: Part 0 - Intro

GPU raytracing can be a fairly opaque and difficult field to understand for the novice. If you are not highly versed in the algorithms of light transport or a seasoned pro at a particular graphics API, trying to get your head around everything at once can feel a bit like drinking from a fire-hose.

A few years back I worked my way through the [Raytracing in One Weekend](https://raytracing.github.io) series of books and found it a lot of fun, as I'm sure have a lot of other people who maybe have never touched raytracing before. I would love to be able to write a similarly accessible introduction to raytracing in Metal. The challenge is that while on the surface, using a raytracing API should make things easier, in practice it is switching out the certainty of mathematical equations for the vagueries of a fairly complex API.

This series is going to be very Mac specific. I am writing in Swift, for macOS, and most likely targetting Apple Silicon. On one hand that probably sets a fairly hefty limit on the target audience, but at the same time it's definitely an under-served market.

In order to follow this series you probably should understand at least the basics of Swift, and a very tiny amount of SwiftUI (although that is only used for a small amount of setup code).

The code for this first part can be found at https://github.com/jonmhobson/SimplestMetalRaytracer. Part 1 will detail the actual parts related to raytracing, but I'll quickly go over the rest of the code here.

## App.swift

The core of the sample is literally just a very simple SwiftUI App that has a single window containing a MetalView

```
import SwiftUI

@main
struct SimplestMetalRaytracerApp: App {
    var body: some Scene {
        Window("SimplestRaytracer", id: "simplest-metal-raytracer") {
            MetalView()
        }
    }
}
```

## MetalView.swift

MTKView is the core view provided to us by MetalKit, and handles a lot of the Metal boilerplate for us. Unfortunately Apple haven't provided a SwiftUI version of this View just yet, so the majority of this code is just wrapping MTKView in an NSViewRepresentable host and setting up our Renderer class, which is where the main rendering logic takes place.

```
import SwiftUI
import MetalKit

class MetalViewInteractor {
    let renderer: Renderer
    let metalView = MTKView()

    init() {
        renderer = Renderer(metalView: metalView)
        metalView.device = renderer.metal.device
        metalView.delegate = renderer
    }
}

struct MetalViewRepresentable: NSViewRepresentable {
    let metalView: MTKView
    func makeNSView(context: Context) -> some NSView { metalView }
    func updateNSView(_ nsView: NSViewType, context: Context) {}
}

struct MetalView: View {
    var interactor = MetalViewInteractor()

    var body: some View {
        MetalViewRepresentable(metalView: interactor.metalView)
    }
}

#Preview {
    MetalView()
}
```

## MetalDevice.swift

Since we're only going to be writing a fairly simple app, we're likely only ever going to have one MTLDevice (our interface to our GPU), one MTLLibrary (which is where our shaders will live), and one MTLCommandQueue (which is the object we use to submit work to the GPU). Therefore we just wrap all these objects up in a single class for convenience.

```
import Foundation
import Metal

class MetalDevice {
    let device: MTLDevice
    let library: MTLLibrary
    let commandQueue: MTLCommandQueue

    init?() {
        guard let device = MTLCreateSystemDefaultDevice(),
              let commandQueue = device.makeCommandQueue(),
              let library = device.makeDefaultLibrary() else {
            return nil
        }

        self.device = device
        self.library = library
        self.commandQueue = commandQueue
    }
}
```

## Renderer.swift

This is where the magic starts to happen. First we initialize our "MetalDevice", and then we create our RaytracingRenderPass (which we will describe later)

```
init(metalView: MTKView) {
    guard let metal = MetalDevice() else { fatalError("Failed to initialize Metal") }
    self.metal = metal
    self.raytracingRenderPass = RaytracingRenderPass(
        metal: metal,
        pixelFormat: metalView.colorPixelFormat
    )
}
```
Next up we have two functions that we need to implement as part of our conformance to MTKViewDelegate. The Renderer was set as the delegate to MTKView, and MTKView will call the first one when our view size changes (which we currently ignore), and then the second function will be called when we want to render (which is by default set at 60 fps, but you can change by setting MTKView's preferredFramesPerSecond variable to your desired value) 

```
func mtkView(_ view: MTKView, drawableSizeWillChange size: CGSize) {}

func draw(in view: MTKView) {
    guard let renderPassDescriptor = view.currentRenderPassDescriptor,
          let commandBuffer = metal.commandQueue.makeCommandBuffer(),
          let renderEncoder = commandBuffer.makeRenderCommandEncoder(descriptor: renderPassDescriptor) else {
        return
    }

    raytracingRenderPass.encode(into: renderEncoder)

    renderEncoder.endEncoding()
    
    if let drawable = view.currentDrawable {
        commandBuffer.present(drawable)
    }

    commandBuffer.commit()
}
```

The second function is a lot more interesting. First we ask our MTKView to give us its currentRenderPassDescriptor, which among other things, contains the render targets that we want to render into.

We then use our commandQueue to create a commandBuffer, and then we use the commandBuffer to create a renderCommandEncoder. You can think of these as representing gradually smaller scopes of GPU work. You will likely have only a single commandQueue for your entire app's lifetime, one commandBuffer per frame, and one or many types of encoder, each representing a particular task.

We then encode our render pass into the render encoder, and then call endEncoding on our render encoder to signify that we're finished with our rendering work - and we must always end our encoders before we can commit them to the GPU.

We then add commands to the commandBuffer to present to the MTKView's drawable, which will be performed after our renderEncoder is finished.

Finally we call commit() to add this commandBuffer to our command queue to be ran on the GPU.

## RaytracingRenderPass.swift

Here is the code with the raytracing code removed (which will be added and explained in part 1)

```
import Foundation
import Metal

class RaytracingRenderPass {
    let renderPipeline: MTLRenderPipelineState

    init(metal: MetalDevice,
         pixelFormat: MTLPixelFormat) {

        let renderDescriptor = MTLRenderPipelineDescriptor()
        renderDescriptor.vertexFunction = metal.library.makeFunction(name: "vertexShader")
        renderDescriptor.fragmentFunction = metal.library.makeFunction(name: "fragmentShader")
        renderDescriptor.colorAttachments[0].pixelFormat = pixelFormat
        self.renderPipeline = try! metal.device.makeRenderPipelineState(descriptor: renderDescriptor)
    }

    func encode(into renderEncoder: MTLRenderCommandEncoder) {
        renderEncoder.setRenderPipelineState(renderPipeline)
        renderEncoder.drawPrimitives(type: .triangle, vertexStart: 0, vertexCount: 6)
    }
}
```

In the init function, we are creating an MTLRenderPipelineDescriptor - telling it where to find the *vertexShader* and *fragmentShader* functions we have created inside our Shaders.metal file.

The rasterization part of this RenderPass is simply rendering a simple 6 vertex quad to cover the entire screen - this is because we want to use the fragment shader to do our raytracing per pixel.

## Shaders.metal

### vertexShader

As you saw in RaytracingRenderPass.swift, we called drawPrimitives with a vertexCount of 6. This means that the following program will be called 6 times, with a vid / vertex_id from 0 to 5.

```
constant float2 quadVertices[] = {
    float2(-1, -1),
    float2(-1,  1),
    float2( 1,  1),
    float2(-1, -1),
    float2( 1,  1),
    float2( 1, -1)
};

struct VertexOut {
    float4 position [[position]];
    float2 uv;
};

vertex VertexOut vertexShader(unsigned short vid [[vertex_id]]) {
    float2 position = quadVertices[vid];

    VertexOut out;
    out.position = float4(position, 0, 1);
    out.uv = position;

    return out;
}

fragment float4 fragmentShader(VertexOut in [[stage_in]]) {
    return float4(1.0);
}
```

The actual vertex positions are given as the constant array *quadVertices*, and we merely use the vertex id to index into this array.

The VertexOut structure is only used to pass data from the Vertex stage into the Fragment shader.

We put the position into the "uv" section, since although it's identical to position, the [[position]] attribute will cause the position to be converted to absolute pixels, while we want to keep a copy in our original range going from -1...1 (also known as NDC / Normalized Device Coordinates)

Right now the fragmentShader simply returns (1, 1, 1, 1) for every pixel.
