# WebGL 1 GPU Particles
This is a proof-of-concept for GPU accelerated particles using WebGL 1 without extensions.

You can see it in action [here](https://jorkov-jac.github.io/projects/gpu-particles)!

## What are GPU Particles?
First, let's cover how simple bouncing particles might be handled on the CPU: You store a series of particles in an array. Each particle stores position information and velocity. Every frame, you loop through the array of particles and update each one, shifting their position based on their velocity and inverting their velocity whenever they reach the edge of the screen. This array is stored in RAM. After updating the particles, you draw them. This means that the CPU (which updates data) needs to send information about the particles to the GPU (which renders the particles), and this can present a bottleneck since the bandwidth between the CPU and GPU is limited. Furthermore, the GPU is designed for processing large amounts of data; wouldn't it be great if we could leverage its parallel processing power to do all of these particle calculations?

Enter GPU particles. Instead of processing particles on the CPU, we get the GPU to store and update the particles itself. And with modern GPUs, we have access to compute shaders which allow us to run such code in a straightforward manner. Great!

## The problem with WebGL
At time of writing, webpages cannot take advantage of compute shaders. Browsers only expose access to WebGL (and more recently, WebGL2), which is very limited compared to what native programs have access to. Whereas compute shaders support general computing, older APIs like WebGL are designed solely for the GPU's original purpose: Drawing stuff. To perform actual calculations with WebGL, we'll need to twist the rendering process into performing computations, which means that we need to work with vertices and textures. But that isn't so bad; vertices are basically just collections of numbers, and since WebGL 2 supports [transform feedback](https://developer.mozilla.org/en-US/docs/Web/API/WebGLTransformFeedback), we can just perform calculations on these vertices directly. It's not too different from using an array.

### I'm not using WebGL2.
That doesn't scratch the itch. I want the full, cursèd experience of doing compuations with colours! So let's limit ourselves to WebGL 1 (without extensions), and store data directly in a texture. We'll only use vertices for rendering the particles themselves. If we need to store position and velocity in 2D then that's 4 numbers, so surely an 8-bit RGBA texture will do; that way every particle is stored as a single pixel in the texture, where a pixel's redness is the particle's X position, greenness its Y position, blueness its X velocity, and opacity its Y velocity.

Nope. There are only 8 bits per channel. That might be fine velocity, but for position? Particles need more than a 256x256 grid, otherwise they need to move really fast (otherwise they won't be able to escape their grid's cell; if they don't have enough speed to leave their current grid position, any incremental progress would get truncated back to 0 and they'd stay in the same place). 16 bits, giving us a 65536x65536 grid, would be better.

So we'll use two adjacent pixels per particle in an 8-bit RGB texture! The left pixel stores the high bytes of the X/Y position as red/green and the X velocity as blue. The right pixel stores the low bytes in red/green and the Y velocity as blue.

For illustration, this data texture is shown as a hypnotic collection of colours in the corner of the screen.

## Terminology
- Shader: A GPU program. There are always two in WebGL; a vertex shader followed by a fragment shader.
- Vertex: Represents the corner of a shape; a vertex shader outputs the corners of a triangle, after which WebGL performs "rasterization" and fills in the triangle using a fragment shader.
- Fragment: Basically a pixel; a fragment shader is a shader that renders pixels (either to a texture or to a screen).

## The process
Every frame:
- Using the current particle data texture, we render a new texture with the updated particle positions and velocities.
    - This is done through a fragment shader; every fragment reads a pair of pixels from the data texture, converts their colours to particle data, updates the particle, and converts the particle's new state back into colours.
- Using the new texture, we render every particle at their position.
    - Every particle is a square, so there are 4 vertices each.
    - Then a fragment shader renders a circular particle within that square.

# Building
Run `npm i` followed by `npx tsc`.
