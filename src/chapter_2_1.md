# Bevy GPU Resources

Bevy wraps a lot of common wgpu GPU resource types to provide additional functionality for bevy. In some cases these types also provide convenience for bevy users. Bevy stores the GPU resources as:

```rust
struct GPUResource {
    id: WrappedID(Uuid), // Note: This is likely to change in the future.
    Arc<WGPUType>
}
```

## Bind Group
Mostly the same as a wgpu bind group however it also includes an ID.

## BufferVec
A buffer vec is a CPU side and GPU side growable array of items on the CPU and a fixed length of items on the GPU. It has the following functions for easy of use:

- `new` - Creates a BufferVec on the CPU side only. `reserve` or `reserve_and_clear` need to be called in order for the buffer to be created on the GPU side as well.
- `staging_buffer` - Returns the temporary staging buffer used to copy data from the CPU to the GPU. 
- `buffer` - The actual wgpu buffer used by the GPU.
- `capacity` - The size limit of the BufferVec
- `push` - Add a new T to the BufferVec. Note: This does not send the CPU data to the GPU and will panic if there is no space left.
- `reserve` - Creates a new buffer and staging buffer with X size while preserving CPU data.
- `reserve_and_clear` - Removes all items in the BufferVec and calls reserve with a new capacity.
- `write_to_staging_buffer` - Writes the data from the CPU to the GPU using a staging buffer. Note: This only writes the CPU data to the staging buffer. Once called you also need to call: `write_to_buffer`.
- `write_to_buffer` - Copies the the data from the Staging Buffer(GPU) to the buffer(GPU).
- `clear` - Clears out the CPU data. Note: Does not clear out the GPU data.

Using a staging buffer allows the actual buffer to be faster because it doesn't require access to the CPU. See: [https://vulkan-tutorial.com/Vertex_buffers/Staging_buffer](https://vulkan-tutorial.com/Vertex_buffers/Staging_buffer)

BufferVec is most likely used for things like vertices/indices buffers.

## Buffer
A simple wrapper around wgpu's buffer type using Arc. Also includes functionality for slicing.

## RenderPipeline
A simple wrapper around wgpu's RenderPipeline type using Arc.

## ComputePipeline
A simple wrapper around wgpu's ComputePipeline type using Arc.

## RenderResourceId
TODO: This likely will change.

## Texture
A simple wrapper around wgpu's texture type. Represents a GPU image. 

- `create_view` - Create's a `TextureView` from the texture.

## TextureView
A simple wrapper around TextureViewValue. Represents part or all of a GPU image. Derefs into a wgpu TextureView.

## TextureViewValue
An enum used by bevy as a way of separating textures and the swap chain textures.

- `TextureView` - A standard texture view.
- `SwapChainFrame` The current frames swap chain frame texture view.

## Sampler
A simple wrapper around wgpu's sampler type. Tells the GPU how to sample a specific texture. I.E. filtering.

## SwapChainFrame
A simple wrapper around wgpu's SwapChainFrame type. Represents the current frame's image that gets displayed.

## UniformVec
A uniform vec is a CPU side and GPU side growable array of items on the CPU and a fixed length of items on the GPU. Similar to the `BufferVec` but it also includes a function for binding to a specific uniform and uses std140 for sizing information.

- `binding` - Gives you a BindingResource that can be used to create the bind group.
