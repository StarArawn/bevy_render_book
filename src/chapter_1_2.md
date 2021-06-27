# GPU Resources
In this section we will cover the types of GPU resources in bevy.

## Textures
TODO: Add back in when bevy's new renderer officially adds textures in.

## Buffers
A buffer is a blob data that lives on the GPU. Buffers are used for anything as simple as a struct or as complex as a graph structure and can be shared across multiple pipelines. You'll send up using buffers quite a bit inside of Bevy. To find out more about buffers visit these links:
- [BufferDescriptor](https://docs.rs/wgpu-types/0.9.0/wgpu_types/struct.BufferDescriptor.html)
- [Buffer](https://docs.rs/wgpu/0.9.0/wgpu/struct.Buffer.html)

CPU data is copied to the buffer either at creation or a variety of other ways:
[queue.write_buffer](https://docs.rs/wgpu/0.9.0/wgpu/struct.Queue.html#method.write_buffer)
[belt.write_buffer](https://docs.rs/wgpu/0.9.0/wgpu/util/struct.StagingBelt.html#method.write_buffer)

Buffers are used to store everything from uniform data to vertex information and even more complex things. Meshes for example can use up to two buffers one for vertices and one for indices. The buffer is automatically created and uploaded to the GPU by the `mesh_resource_provider_system` system when the asset system loads in a new mesh or a change is detected. Here is some example code for creating a buffer:

```rust
let vertex_buffer = Buffer::from(render_device.create_buffer_with_data(
    &BufferInitDescriptor {
        usage: BufferUsage::VERTEX,
        label: None,
        contents: &vertex_buffer_data,
    },
));
```

## 