# FromWorld, Extract, Prepare, Queue, and Draw
In this section we'll cover the 4 main stages of Bevy's renderer.
1. [Building the shaders, pipelines, and bind group layouts using `FromWorld`.](#1-building-the-shaders-pipelines-and-bind-group-layouts-using-fromworld)
2. [Extract Stage: GPU data from ECS data that will be consumed by Prepare/Draw.](#2-extract-stage-gpu-data-from-ecs-data-that-will-be-consumed-by-preparedraw)
3. [Prepare Stage: Prepares the GPU data by copying the data from the CPU to the GPU.](#3-prepare-stage-prepares-the-gpu-data-by-copying-the-data-from-the-cpu-to-the-gpu)
4. [Queue Stage: Creating bind groups and draw calls from extracted data.](#4-queue-stage-creating-bind-groups-and-draw-calls-from-extracted-data)
5. [Draw Stage: Draw by setting pipelines, bindings, buffers, and calling draw commands.](#5-draw-stage-draw-by-setting-pipelines-bindings-buffers-and-calling-draw-commands)

### Bevy Render Stages:
- Extract - Extract data from "app world" and insert it into "render world". This step should be kept as short as possible to increase the "pipelining potential" for running the next frame while rendering the current frame.
- Prepare - Prepare render resources from extracted data.
- Queue - Create Bind Groups that depend on Prepare's data and queue up draw calls to run during the Render stage.
- PhaseSort - Sort RenderPhases here
- Render - Actual rendering happens here. In most cases, only the render backend should insert resources here.
- Cleanup - Clean up render resources here.

Some of these stages might be changed, And perhaps some new ones would be added in the future.

<!-- TODO: Replace this section when the time comes. -->
## 1. Building the shaders, pipelines, and bind-group layouts using `FromWorld`.

`FromWorld` is a trait provided by Bevy that allows you to directly access the World when initializing a resource for the first time. In the future, loading shaders/pipelines/layouts using `FromWorld` will be replaced by the asset system.

Example:

```rust
pub struct MyResource;

impl FromWorld for MyResource {
    fn from_world(world: &mut World) -> Self {
        // Do stuff with world here and return a Bevy resource.
    }
}
```

Using this trait we can build out our render/compute pipelines and bind-group layouts. Example:
```rust
    // Grab the render device from world.
    let render_device = world.get_resource::<RenderDevice>().unwrap();

    // First load in the shaders for the pipeline.
    let vertex_shader = Shader::from_glsl(ShaderStage::VERTEX, include_str!("my_shader.vert"))
        .get_spirv_shader(None)
        .unwrap();
    let fragment_shader = Shader::from_glsl(ShaderStage::FRAGMENT, include_str!("my_shader.frag"))
        .get_spirv_shader(None)
        .unwrap();

    // Next we build the shader modules from the shaders:
    // Note: Most of the stuff here is very similar to how its currently done in wgpu.
    let vertex_spirv = vertex_shader.get_spirv(None).unwrap();
    let fragment_spirv = fragment_shader.get_spirv(None).unwrap();

    let vertex_shader_module = render_device.create_shader_module(&ShaderModuleDescriptor {
        flags: ShaderFlags::default(),
        label: None,
        source: ShaderSource::SpirV(Cow::Borrowed(&vertex_spirv)),
    });
    let fragment_shader_module = render_device.create_shader_module(&ShaderModuleDescriptor {
        flags: ShaderFlags::default(),
        label: None,
        source: ShaderSource::SpirV(Cow::Borrowed(&fragment_spirv)),
    });


    // Create our bind-group layouts for the shader
    // In this example we'll just have 1 layout which represents
    // our view matrix.
    let view_layout = render_device.create_bind_group_layout(&BindGroupLayoutDescriptor {
        entries: &[
            // View
            BindGroupLayoutEntry {
                binding: 0,
                visibility: ShaderStage::VERTEX | ShaderStage::FRAGMENT,
                ty: BindingType::Buffer {
                    ty: BufferBindingType::Uniform,
                    has_dynamic_offset: true,
                    // The size of a single matrix.
                    min_binding_size: BufferSize::new(Mat4::std140_size_static() as u64),
                },
                count: None,
            },
        ],
        label: None,
    });

    // Now create the pipeline layout again, exactly the same you would with wgpu.
    let pipeline_layout = render_device.create_pipeline_layout(&PipelineLayoutDescriptor {
        label: None,
        push_constant_ranges: &[],
        bind_group_layouts: &[&view_layout],
    });

    // Now, we can create our vertex state:
    let vertex_state = VertexState {
        buffers: &[VertexBufferLayout {
            array_stride: 32, // The total byte size of a single vertex.
            step_mode: InputStepMode::Vertex,
            attributes: &[
                // Because Bevy meshes sort attributes alphabetically the first attribute
                // is actually Normal. Notice the offsets are different, but we want the
                // shader location to be 0. After all position, normal, uv is fairly
                // standard.
                VertexAttribute {
                    format: VertexFormat::Float32x3,
                    offset: 12, // Offset in the vertex in bytes
                    // Location index which corresponds to a layout location in the shader.
                    shader_location: 0,
                },
                // Normal
                VertexAttribute {
                    format: VertexFormat::Float32x3,
                    offset: 0,
                    shader_location: 1,
                },
                // Uv
                VertexAttribute {
                    format: VertexFormat::Float32x2,
                    offset: 24,
                    shader_location: 2,
                },
            ],
        }],
        module: &&vertex_shader_module,
        entry_point: "main",
    };

    // Create fragment state.
    let fragment_state = FragmentState {
        module: &&fragment_shader_module,
        entry_point: "main",
        // Information about the target texture the fragment is writing to.
        targets: &[ColorTargetState {
            // Default frame buffer texture Bevy uses.
            format: TextureFormat::bevy_default(), 
            // Default blend states.
            blend: Some(BlendState {
                color: BlendComponent {
                    src_factor: BlendFactor::SrcAlpha,
                    dst_factor: BlendFactor::OneMinusSrcAlpha,
                    operation: BlendOperation::Add,
                },
                alpha: BlendComponent {
                    src_factor: BlendFactor::One,
                    dst_factor: BlendFactor::One,
                    operation: BlendOperation::Add,
                },
            }),
            write_mask: ColorWrite::ALL,
        }],
    };

    // Create depth state.
    let depth_stencil_state = DepthStencilState {
        // Default Bevy depth format.
        format: TextureFormat::Depth32Float,
        depth_write_enabled: true,
        depth_compare: CompareFunction::Less,
        stencil: StencilState {
            front: StencilFaceState::IGNORE,
            back: StencilFaceState::IGNORE,
            read_mask: 0,
            write_mask: 0,
        },
        bias: DepthBiasState {
            constant: 0,
            slope_scale: 0.0,
            clamp: 0.0,
        },
    };

    // Create primitive state.
    let primitive_state = PrimitiveState {
        topology: PrimitiveTopology::TriangleList,
        strip_index_format: None,
        front_face: FrontFace::Ccw,
        cull_mode: Some(Face::Back),
        polygon_mode: PolygonMode::Fill,
        clamp_depth: false,
        conservative: false,
    };

    // Finally create the pipeline itself.
    let pipeline = render_device.create_render_pipeline(&RenderPipelineDescriptor {
        label: None,
        vertex: vertex_state,
        fragment: Some(fragment_state),
        depth_stencil: Some(depth_stencil_state),
        layout: Some(&pipeline_layout),
        multisample: MultisampleState::default(),
        primitive: primitive_state,
    });
```
A lot of the code in this section wasn't explained. For a more detailed explanation, please see the resource links in the [introduction](./introduction.md#additional-resources).


## 2. Extract Stage: Extract GPU data from Bevy ECS data that will be consumed by the Prepare/Draw stages.
Bevy's rendering system relies on this stage to extract any ECS data that should be given to the GPU. These could be things like meshes, uniforms, etc. 

First, Bevy expects some sort of extracted data type. Here is an example of what sprites look like:

```rust
struct ExtractedSprite {
    transform: Mat4,
    size: Vec2,
    handle: Handle<Image>,
}
```

Notice only the data being passed to the GPU lives inside of this struct. `Bevy_sprite` also creates a resource called `ExtractedSprites` which contains a vec of all of the extracted data. Something like `Vec<ExtractedSprite>`.

Then, we write a system that queries the data that we want and inserts it into the resource:
```rust
pub fn extract_sprites(
    mut commands: Commands,
    images: Res<Assets<Image>>,
    query: Query<(&Sprite, &GlobalTransform, &Handle<Image>)>,
) -> Self {
    let mut extracted_sprites = Vec::new();
    for (sprite, transform, image_handle) in query.iter() {
        // Skip over sprites that have no loaded-in image assets.
        if !images.contains(image_handle) {
            continue;
        }

        // Push extracted sprite data into our vec.
        extracted_sprites.push(ExtractedSprite {
            transform: transform.compute_matrix(),
            size: sprite.size,
            handle: image_handle.clone_weak(),
        })
    }

    // Finally insert the extracted sprites as a resource:
    commands.insert_resource(ExtractedSprites {
        sprites: extracted_sprites,
    });
}
```

That's it for extraction. Obviously this is a simple example. A more complex example might cache data, and use a more complicated resource format aside from a `Vec`. For simple sprite rendering, a regular Vec works quite well.

## 3. Prepare Stage: Prepares the GPU data by copying the data from the CPU to the GPU.
In this stage we pass data from the CPU to the GPU by using various commands. See the chapter on GPU resources for a hint at how to push data from the CPU to the GPU. Bevy provides some types to make this easier.

Setting up the "Meta" data which is passed to the Queue stage:
```rust
// Note: This is taken directly from `Bevy_sprite` with added explanation comments.
#[repr(C)]
#[derive(Copy, Clone, Pod, Zeroable)]
struct SpriteVertex {
    pub position: [f32; 3],
    pub uv: [f32; 2],
}

pub struct SpriteMeta {
    // Our vec of sprite vertices. Notice how we use a BufferVec here
    // which provides some convenience for taking CPU data and passing it to the GPU.
    vertices: BufferVec<SpriteVertex>,
    
    // A vec of our indices, again using a BufferVec like above.
    indices: BufferVec<u32>,
    
    // A Bevy mesh representing a quad created using:
    // Quad {
    //  size: Vec2::new(1.0, 1.0),
    //  ..Default::default()
    // }
    quad: Mesh,
    
    // A bind group for the view matrix.
    // Will be explained in more detail in the Queue stage's section.
    view_bind_group: Option<BindGroup>,
    
    // Bind groups for the textures.
    // Will be explained in more detail in the queue section.
    texture_bind_groups: Vec<BindGroup>,

    // A map of images to texture bind-group ids.
    // This is used later in the Queue step to keep track of bindings
    // that have already been created for a particular image.
    texture_bind_group_indices: HashMap<Handle<Image>, usize>,
}
```
The metadata is created as a resource using: `init_resource::<SpriteMeta>()` and as such it requires implementing the Default trait.


Example System:
```rust
fn prepare_sprites(
    render_device: Res<RenderDevice>,
    mut sprite_meta: ResMut<SpriteMeta>,
    extracted_sprites: Res<ExtractedSprites>,
) {
    // If we have no extract sprites, jump out early..
    if extracted_sprites.sprites.len() == 0 {
        return;
    }

    // Grab our vertex-positions for the quad mesh.
    let quad_vertex_positions = if let VertexAttributeValues::Float32x3(vertex_positions) =
        sprite_meta
            .quad
            .attribute(Mesh::ATTRIBUTE_POSITION)
            .unwrap()
            .clone()
    {
        vertex_positions
    } else {
        panic!("expected vec3");
    };

    // Grab our vertex-uvs for the quad mesh.
    let quad_vertex_uvs = if let VertexAttributeValues::Float32x2(vertex_uvs) = sprite_meta
        .quad
        .attribute(Mesh::ATTRIBUTE_UV_0)
        .unwrap()
        .clone()
    {
        vertex_uvs
    } else {
        panic!("expected vec2");
    };

    // Grab the quad mesh's indices.
    let quad_indices = if let Indices::U32(indices) = sprite_meta.quad.indices().unwrap() {
        indices.clone()
    } else {
        panic!("expected u32 indices");
    };

    // Create our vertex buffer by reserving(create a new buffer on the GPU)
    // and clearing(removing the data on the CPU).
    // Note the size of the vertex buffer is (sprite_count * quad_vertices_count).
    sprite_meta.vertices.reserve_and_clear(
        extracted_sprites.sprites.len() * quad_vertex_positions.len(),
        &render_device,
    );
    // Similar to vertices we are creating a new buffer on the GPU.
    // Note the size of the indices buffer is (sprite_count * quad_indices_count)
    sprite_meta.indices.reserve_and_clear(
        extracted_sprites.sprites.len() * quad_indices.len(),
        &render_device,
    );

    // For each extracted sprite create a set of quad vertices and indices by
    // filling up the buffers we created above.
    for (i, extracted_sprite) in extracted_sprites.sprites.iter().enumerate() {
        for (vertex_position, vertex_uv) in quad_vertex_positions.iter().zip(quad_vertex_uvs.iter())
        {
            // Calculate the scaled quad positions.
            let mut final_position =
                Vec3::from(*vertex_position) * extracted_sprite.size.extend(1.0);
            // Make sure the transform is applied to the position.
            final_position = (extracted_sprite.transform * final_position.extend(1.0)).xyz();
            // Push a vertex into our vertex buffer.
            sprite_meta.vertices.push(SpriteVertex {
                position: final_position.into(),
                uv: *vertex_uv,
            });
        }

        for index in quad_indices.iter() {
            // Indices are much simpler just push the indices straight in.
            sprite_meta
                .indices
                .push((i * quad_vertex_positions.len()) as u32 + *index);
        }
    }

    // Finally push the data from the CPU into the GPU staging buffer.
    // A staging buffer is used to increase performance as GPU buffers
    // that have CPU access are much slower.
    sprite_meta.vertices.write_to_staging_buffer(&render_device);
    sprite_meta.indices.write_to_staging_buffer(&render_device);
}
```

One final step is required in order to push our buffers to the GPU. This likely will get moved into the prepare stage at some point in the future.
<!-- TODO: Remove and update the prepare stage when that happens.. -->
```rust
// See the section on nodes and the graph for more information about nodes.
impl Node for SpriteNode {
    fn run(
        &self,
        _graph: &mut RenderGraphContext,
        render_context: &mut RenderContext,
        world: &World,
    ) -> Result<(), NodeRunError> {
        // Grab a reference to the sprite meta data.
        let sprite_buffers = world.get_resource::<SpriteMeta>().unwrap();
        // Copy the vertex staging buffers(GPU) to the vertex buffers(GPU).
        sprite_buffers
            .vertices
            .write_to_buffer(&mut render_context.command_encoder);
        // Copy the indices staging buffers(GPU) to the indices buffers(GPU).
        sprite_buffers
            .indices
            .write_to_buffer(&mut render_context.command_encoder);
        Ok(())
    }
}
```

## 4. Queue Stage: Creating bind groups and draw calls from extracted data.
In this final stage before drawing, we need to create bind groups, and build out our draw calls.

### The Drawable struct
Bevy uses a Drawable struct to represent draw calls. The struct looks like:

```rust
pub struct Drawable {
    pub draw_function: DrawFunctionId,
    pub draw_key: usize,
    pub sort_key: usize,
}
```

- `draw_function` is an ID to the actual function which does the drawing. We cover this in the next [section]() below.
- `draw_key` is just an index per draw call(I.E. an index to the sprite). Later this draw key is used to tell the draw commands which sprite is drawing.
- `sort_key` the sort key represents an index to the binding collection. This part is a bit harder to explain, but imagine you have two sprites. Sprite 1 has a texture A, and sprite 2 has a texture B. For each texture, we have a separate bind group, and as such resulting index for that bind group. So, sprite 1 has a `sort_key` of 1, and sprite 2, has a `sort_key` of 2. If they both shared the same texture, the `sort_key` would be 1 for both drawable instances. For more complex materials, a more complex sort key is required. Bevy requires a sort key so it can efficiently render your draw calls in an order that requires the fewest state changes between them. Every switch of pipelines, vertex buffers, or bind groups is a costly operation on the GPU. By limiting the amount of switching required by sorting like-items we can remove a lot of unnecessary state changes.

Draw calls seem fairly easy to create so far, but where do we put the draw calls once we've created them? The Bevy renderer has RenderPhases which help out with this. Currently Bevy has the following render phases:

- Transparent2dPhase
- Transparent3dPhase

These phases are used by the `MainPass2dNode` and the `MainPass3dNode` to draw stuff to the screen. In the future we can expect to likely see more render phases as well as more pass nodes. For example a full-screen pass node can be expected which would be used for rendering post-processing effects.

`RenderPhase` itself is quite simple, being a storage container for draw calls(`Drawable`) that also sorts the draw calls by their sort key.

### Creating bind groups
During the queue step, one responsibility is to collect any assets/resources/etc and create bind groups with that data. For sprites in Bevy, this means locating the sprite texture, building a bind-group for each texture, and a resulting sort-key for said sprite. If you remember from before in the prepare step, our sprite meta-data has two additional fields called: `texture_bind_groups` and `texture_bind_group_indices`. When we create a bind group for a texture, we'll insert that bind group into `texture_bind_groups`. We'll also insert the texture used into the `texture_bind_group_indices` HashMap where the key is the handle to the image and the value is the index pointing to the bind group in `texture_bind_groups`. We only want to create 1 bind group per texture, and not a bind group per sprite. in the following code example you'll see how we can leverage the HashMap in order to optimize our bind groups. Finally, we also have a `view_bind_group` which will contain our bind-group for the camera's data.

### Example

```rust
pub fn queue_sprites(
    // A resource containing all of the draw call functions.
    // This is explained in more detail in the next section.
    draw_functions: Res<DrawFunctions>,
    // The wgpu device wrapper Bevy provides which is used to create bind groups.
    render_device: Res<RenderDevice>,
    // Our sprite meta-data from the previous step.
    mut sprite_meta: ResMut<SpriteMeta>,
    // Bevy's default camera-view meta-data.
    // This is a great example of how we can use data from other rendering plugins.
    // Here we have meta-data that Bevy creates for the active camera.
    // Since the meta-data is created in Prepare we can be assured that the
    // data will be available in Queue even across rendering plugins.
    view_meta: Res<ViewMeta>,
    // Our sprite shaders resource which contains our bind-group layouts.
    // These were created in the first step.
    sprite_shaders: Res<SpriteShaders>,
    // The extracted sprites from the Extract step.
    extracted_sprites: Res<ExtractedSprites>,
    // A list of all image(texture) assets.
    gpu_images: Res<RenderAssets<Image>>,
    // Our render phase
    mut views: Query<&mut RenderPhase<Transparent2dPhase>>,
) {
    // Create the bind group for the view first.
    sprite_meta.view_bind_group.get_or_insert_with(|| {
        render_device.create_bind_group(&BindGroupDescriptor {
            entries: &[BindGroupEntry {
                binding: 0, // Binding number here should match our shader binding.
                resource: view_meta.uniforms.binding(),
            }],
            label: None,
            layout: &sprite_shaders.view_layout,
        })
    });
    
    let sprite_meta = &mut *sprite_meta;
    // Pull out the draw sprites function from the draw_functions resource.
    let draw_sprite_function = draw_functions.read().get_id::<DrawSprite>().unwrap();
    // Iterate over each render phase. In this example there is only one phase.
    for mut transparent_phase in views.iter_mut() {
        let texture_bind_groups = &mut sprite_meta.texture_bind_groups;
        // Loop through each sprite in the extracted sprites collection and enumerate on them. Their 
        // index will become our draw_key.
        for (i, sprite) in extracted_sprites.sprites.iter().enumerate() {
            // Now we either create the bind group if it doesn't exist.
            // OR we pull the bind-group index from the texture_bind_group_indices
            // HashMap.
            let bind_group_index = *sprite_meta
                .texture_bind_group_indices
                .entry(sprite.handle.clone_weak())
                .or_insert_with(|| {
                    // Find our GPU image.
                    let gpu_image = gpu_images.get(&sprite.handle).unwrap();
                    // Create a new index for the bind group. The index points to a bind group
                    // inside of texture_bind_groups.
                    let index = texture_bind_groups.len();
                    // Now create the bind group. Use the texture view and sampler from
                    // the gpu_image.
                    let bind_group = render_device.create_bind_group(&BindGroupDescriptor {
                        entries: &[
                            BindGroupEntry {
                                binding: 0,
                                resource: BindingResource::TextureView(&gpu_image.texture_view),
                            },
                            BindGroupEntry {
                                binding: 1,
                                resource: BindingResource::Sampler(&gpu_image.sampler),
                            },
                        ],
                        label: None,
                        layout: &sprite_shaders.material_layout,
                    });
                    // Finally we push the bind group into the vec.
                    texture_bind_groups.push(bind_group);
                    // And return the index we created earlier.
                    index
                });
            
            // Once we have the bind_group_index we can create our draw call.
            transparent_phase.add(Drawable {
                // The function used to draw.
                draw_function: draw_sprite_function,
                // Our draw key which in this case is just the sprite index.
                draw_key: i,
                // Our sort key which is the bind-group index.
                // Render phase will later sort the drawables using this key.
                sort_key: bind_group_index,
            });
        }
    }
}
```
And that wraps up Queue. The reason Queue feels more complex than some of the other steps is because of the need for sort-keys to optimize our draw calls. Hopefully it's clear by now how that can be relatively simple. It's neat to see how we can bring in data from other rendering plugins and use it in our plugin with minimal issues.

## 5. Draw Stage: Draw by setting pipelines, bindings, buffers, and calling draw commands.
To create a drawable, a draw function is required. The draw function tells Bevy what pipelines, bind groups, vertex buffers, and what draw commands to use. 

Example:
```rust
// First we need to create a query to locate the data:
type DrawSpriteQuery<'s, 'w> = (
    // Our pipeline is stored in here from the first step.
    Res<'w, SpriteShaders>,
    // Our sprite meta data from prepare.
    Res<'w, SpriteMeta>,
    // Our view uniform offset which is used to grab the camera data.
    Query<'w, 's, &'w ViewUniformOffset>,
);

// The start of creating our draw function.
pub struct DrawSprite {
    // Our DrawSpriteQuery is stored on the struct inside of some system state.
    // TODO: explain some of this a bit more in depth.
    params: SystemState<DrawSpriteQuery<'static, 'static>>,
}

impl DrawSprite {
    // Create our DrawSprite with the SystemState that holds our query.
    pub fn new(world: &mut World) -> Self {
        Self {
            params: SystemState::new(world),
        }
    }
}

// Now we implement the Draw trait for our DrawSprite
impl Draw for DrawSprite {
    fn draw<'w>(
        &mut self,
        // The Bevy world
        world: &'w World,
        // The current render pass that has some internal state.
        pass: &mut TrackedRenderPass<'w>,
        // The entity for the current view camera.
        view: Entity,
        // The draw key for this drawable.
        draw_key: usize,
        // The sort key for this drawable.
        sort_key: usize,
    ) {
        const INDICES: usize = 6;
        // Grab the pipeline, sprite metadata and view information.
        let (sprite_shaders, sprite_meta, views) = self.params.get(world);
        // Grab the current view uniform.
        let view_uniform = views.get(view).unwrap();
        // Sprite metadata.
        let sprite_meta = sprite_meta.into_inner();
        // Set the pipeline.
        pass.set_render_pipeline(&sprite_shaders.into_inner().pipeline);
        // Set the vertex buffer.
        pass.set_vertex_buffer(0, sprite_meta.vertices.buffer().unwrap().slice(..));
        // Set the index buffer.
        pass.set_index_buffer(
            sprite_meta.indices.buffer().unwrap().slice(..),
            0,
            IndexFormat::Uint32,
        );
        // Set the bind group for the view.
        pass.set_bind_group(
            0,
            sprite_meta.view_bind_group.as_ref().unwrap(),
            &[view_uniform.offset],
        );

        // Set the bind group for the texture by using the sort key to look up the
        // bind groups we created in the Queue step.
        pass.set_bind_group(1, &sprite_meta.texture_bind_groups[sort_key], &[]);

        // Finally draw our sprite using the draw_key as an index into the vertex/indices
        // buffer. draw_indexed is like wgpu's draw_indexed.
        pass.draw_indexed(
            (draw_key * INDICES) as u32..(draw_key * INDICES + INDICES) as u32,
            0,
            0..1,
        );
    }
}
```

If you'll notice, at first glance it appears as though we are resetting the pipelines/bind groups/etc for each draw call. Instead Bevy uses a `TrackedRenderPass` which has state that won't allow the same thing to be set twice. Example:
```rust
    pub fn set_render_pipeline(&mut self, pipeline: &'a RenderPipeline) {
        debug!("set pipeline: {:?}", pipeline);
        // Check the state to see if we've already 
        // set this pipeline last. If true, we return early.
        if self.state.is_pipeline_set(pipeline.id()) {
            return;
        }
        // Otherwise set the pipeline and update the state..
        self.pass.set_pipeline(pipeline);
        self.state.set_pipeline(pipeline.id());
    }
```
