# Render Plugins
When bevy first creates an app the default render plugin is added. Next a new sub app is created for the entire render plugin. A WGPU render device is created along with a queue and other WGPU related resources which are added to the app as standard bevy_ecs resources. As we learned from the previous [section](./chapter_2_2.md) there are several rendering stages these stages get created now.  The render graph and draw functions resources are created next. 

The sub app is specifically built to contain any and all rendering data. During the extract stage of the rendering the sub app is temporarily linked to the world in the main app. At the end of each frame all of the entities in the rendering sub app are cleared out.

If you would like to add your own rendering plugin(custom shader, pass, etc) you can easily do so by using a new rendering plugin. We'll use `bevy_sprite` as an example on how we can do this. If you are not familiar with bevy's plugin system I highly recommend you read up on it. You can find more information here: [Bevy Plugins](https://bevy-cheatbook.github.io/programming/plugins.html) <!-- TODO: Replace this link with one from the official book..-->

Example:
```rust
// First create an empty struct which serves as the "plugin".
#[derive(Default)]
pub struct SpritePlugin;

// Next we implement Plugin.
impl Plugin for SpritePlugin {
    fn build(&self, app: &mut App) {
        // Register any types we have.
        app.register_type::<Sprite>();
        // We want a mutable reference to the sub app.
        // This is used to change the sub app and add our own
        // custom systems that follow the different render stages.
        let render_app = app.sub_app_mut(0);
        render_app
            // Add in our Extract stage system.
            .add_system_to_stage(RenderStage::Extract, render::extract_sprites.system())
            // Add in the Prepare stage system.
            .add_system_to_stage(RenderStage::Prepare, render::prepare_sprites.system())
            // Add in the Queue stage system.
            .add_system_to_stage(RenderStage::Queue, queue_sprites.system())
            // Add in our resources for pipelines and meta data.
            .init_resource::<SpriteShaders>()
            .init_resource::<SpriteMeta>();
        // Add our draw sprite function to the DrawFunctions resource.
        let draw_sprite = DrawSprite::new(&mut render_app.world);
        render_app
            .world
            .get_resource::<DrawFunctions>()
            .unwrap()
            .write()
            .add(draw_sprite);
        // Finally we need mutable access to the render app's world.
        let render_world = app.sub_app_mut(0).world.cell();
        // This is used to insert our "node" for sprites into the render graph.
        let mut graph = render_world.get_resource_mut::<RenderGraph>().unwrap();
        graph.add_node("sprite", SpriteNode);
        // We also add a node edge which is used to tell bevy that this node should run
        // before the main pass. This is because main pass is what actually runs our draw
        // call functions. 
        graph
            .add_node_edge("sprite", core_pipeline::node::MAIN_PASS_DEPENDENCIES)
            .unwrap();
    }
}
```

As you can see the sprite render plugin is pretty simple. More complex shaders which have lots of inputs might be more complex and need more information here, but the core concept should be the same.