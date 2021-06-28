# Bevy GPU Resources

Bevy wraps many of wgpu's common GPU resource types, and adds to some of them additional functionality to provide convenience for Bevy users. Bevy stores the GPU resources as:

```rust
struct GPUResource {
    id: WrappedID(Uuid), // Note: This is likely to change in the future.
    Arc<WGPUType>
}
```

## Bind Group
Mostly the same as wgpu's bind groups, however, it also includes an ID.

## BufferVec
A buffer vec on the CPU-side is a growable array of items. On the GPU-side, a buffer vec has a fixed length. It has the following functions for ease of use:

- `new` - Creates a BufferVec on the CPU side only. `reserve` or `reserve_and_clear` need to be called in order for the buffer to be created on the GPU's side as well.
- `staging_buffer` - Returns the temporary staging buffer used to copy data from the CPU to the GPU. 
- `buffer` - The actual wgpu buffer used by the GPU.
- `capacity` - The size limit of the BufferVec
- `push` - Adds a new T to the BufferVec. Note: This does not send the CPU data to the GPU, and will panic if you try to use it when the buffer's capacity is full.
- `reserve` - Creates a new buffer and staging buffer with `new_capacity` size on the GPU side, while preserving the CPU-side data.
- `reserve_and_clear` - Removes all items in the BufferVec and calls reserve with a new capacity.
- `write_to_staging_buffer` - Writes the data from the CPU to the GPU using a staging buffer. Note: This only writes the CPU data to the staging buffer. Once called you also need to call: `write_to_buffer`.
- `write_to_buffer` - Copies the the data from the Staging Buffer(GPU) to the buffer(GPU).
- `clear` - Clears out the CPU data. Note: Does not clear out the GPU data.

Using a staging buffer allows the actual buffer to be faster because it doesn't require access to the CPU. See: [https://vulkan-tutorial.com/Vertex_buffers/Staging_buffer](https://vulkan-tutorial.com/Vertex_buffers/Staging_buffer)

BufferVec is most likely used for things like vertex buffers and index buffers.

## Buffer
A simple wrapper around wgpu's buffer type using an Arc. Also includes functionality for slicing.

## RenderPipeline
A simple wrapper around wgpu's RenderPipeline type using an Arc.

## ComputePipeline
A simple wrapper around wgpu's ComputePipeline type using an Arc.

## RenderResourceId
TODO: This likely will change.

## Texture
A simple wrapper around wgpu's texture type. Represents a GPU image. 

- `create_view` - Create's a `TextureView` from the texture.

## TextureView
A simple wrapper around TextureViewValue. Represents part or all of a GPU image. Derefs into a wgpu TextureView.

## TextureViewValue
An enum used by Bevy to separate textures and swap-chain frames.

- `TextureView` - A standard texture view.
- `SwapChainFrame` The current swap-chain-frame's texture view.

## Sampler
A simple wrapper around wgpu's sampler type. Tells the GPU how to sample a specific texture. I.E. filtering.

## SwapChainFrame
A simple wrapper around wgpu's SwapChainFrame type. Represents the current frame's displayed image.

## UniformVec
A uniform vec on the CPU-side is a growable array of items. On the GPU-side, a uniform vec has a fixed length.
Similar to the `BufferVec` but it also includes a function for binding to a specific uniform and uses std140 for sizing information.

- `binding` - Gives you a BindingResource that can be used to create the bind group.
