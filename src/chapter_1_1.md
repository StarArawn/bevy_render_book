# Pipelines and Bindings

Bevy, like many other game engines and or API's, has a concept of a rendering pipeline. A render pipeline's sole job is to provide information to the GPU about how and what it is rendering.

Bevy has two pipelines, a render pipeline, and a compute pipeline. In the future, there might be more pipelines created to provide additional functionality. In simple terms, a pipeline, is a CPU representation of the
shader you are using, along with any additional information the GPU needs to know. 

## Render Pipelines
Render pipelines contain: vertex state, fragment state, depth state, shader layout-definitions, and shader-modules. These are used to tell the GPU what you are rendering, and how to render it. You can see their documentation here:
- [RenderPipelineDescriptor](https://docs.rs/wgpu/0.9.0/wgpu/struct.RenderPipelineDescriptor.html)

## Compute Pipelines
Compute pipelines are much simpler because of their general-purpose nature. They contain only the layout of the shader and the shader-module. You can see their documentation here:
- [ComputePipelineDescriptor](https://docs.rs/wgpu/0.9.0/wgpu/struct.ComputePipelineDescriptor.html)

## Bind Groups and Layouts
Bind-group layouts are descriptions of GPU resources, used by the shader, along with how the GPU resources are accessed. Bind groups are created from bind-group layouts and specific resources. GPU resources can be things like: textures, uniforms, etc. 

Important reading:
- [BindGroupLayoutDescriptor](https://docs.rs/wgpu/0.9.0/wgpu/struct.BindGroupLayoutDescriptor.html)
- [BindGroupLayout](https://docs.rs/wgpu/0.9.0/wgpu/struct.BindGroupLayout.html)
- [BindGroupDescriptor](https://docs.rs/wgpu/0.9.0/wgpu/struct.BindGroupLayout.html)
- [BindGroup](https://docs.rs/wgpu/0.9.0/wgpu/struct.BindGroup.html)

