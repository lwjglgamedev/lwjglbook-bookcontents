# Chapter 20 - Indirect drawing (static models)

Until this chapter, we have rendered the models by binding their material uniforms, their textures, their vertices and indices buffers and submitting one draw command for each of the meshes they are composed. In this chapter, we will start our way to a more efficient wat of rendering, we will begin the implementation of a bind-less render (at aleast almost bind-less). In this type of rendering we do not invoke a bunch of draw commands to draw the scene, instead we populate a buffer with the instructions that will allow the GPU to render them. This is called indirect rendering and it is a more efficient way of drawing because:

- We remove the need to perform several bind operations before drawing each mesh.
- We need just to invoke a single draw call.
- We can perform in-GPU operations, such as frustum culling reducing the load on the CPU side.

As you can see, the ultimate goal is to maximize the utilization of the GPU while removing potential bottlenecks that may occur at the CPU side and latencies due to CPU to GPU communications. In this chapter we will transform our render to use indirect drawing starting with just static models. Animated models will be handled in next chapter.

You can find the complete source code for this chapter [here](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-20).

## Concepts

Prior to explaining the code, let's explain the concepts behind indirect drawing.

[Next chapter](../chapter-21/chapter-21.md)