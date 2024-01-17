### GPUIO

GPUIO is a project in the domain of computer graphics and GPU computing, specializing in physics simulations. It harnesses the capabilities of modern GPUs, supporting WebGL 1/2 and GLSL 1/3, and offers the following simulation types:

1. Grid-Based Simulation: Explores dynamic patterns, including Reaction Diffusion.

2. Particle + Grid Simulations: Combines particles and grids for intricate simulations, such as Net, Dots, Honeycomb, Fingerprints, and Fibers.

3. 2D Wave Equation Simulation: Analyzes wave propagation in a 2D space.

4. Conway's Game of Life: Demonstrates emergent behavior through cellular automata.

5. Julia Set Fractal: Creates intricate fractal art using complex mathematics.


### Motivation

The main motivation behind gpu-io is to make it easier to compose GPU-accelerated applications without worrying too much about low-level WebGL details.  This library manages WebGL state, implements shader and program caching, and deals with issues of available WebGL versions or spec inconsistencies across different browsers/hardware.  It should significantly cut down on the amount of boilerplate code and state management you need to do in your applications.  At the same time, gpu-io gives you enough low-level control to write extremely efficient programs for computationally demanding applications.

[As of Feb 2022, WebGL2 has now been rolled out to all major platforms](https://www.khronos.org/blog/webgl-2-achieves-pervasive-support-from-all-major-web-browsers) (including mobile Safari and Microsoft Edge) - but even among WebGL2 implementations, there are differences in behavior across browsers (especially mobile).  Additionally, you may still come across non-WebGL2 enabled browsers in the wild for some time.  gpu-io rigorously checks for these gotchas and uses software polyfills to patch any issues so you don't have to worry about it.  gpu-io will also attempt to automatically convert your GLSL3 shader code into GLSL1 so that it can run in WebGL1 in a pinch. See tests/README.md for more information on browser support.

- [Installation](#installation)
- [Use](#use)
- [Examples](#examples)
- [API](#api)
- [Compatibility with Threejs](#compatibility-with-threejs)
- [Limitations/Notes](#limitationsnotes)
- [Acknowledgements](#acknowledgements)
- [License](#license)
- [Development](#development)

### Grid-Based Simulations : a reaction-diffusion system, where two virtual chemicals interact to create dynamic patterns. Similar systems were proposed by Alan Turing in 1952 as a mechanism for pattern formation in plants and animals. This reaction-diffusion system is particularly sensitive to two parameters: K (varies along x-axis) and F (varies along y-axis). You can interactively explore regions of K,F parameter space by zooming and panning across the screen.

### Experience the Reaction Diffusions simulation's dynamics in this illuminating video :

https://github.com/Malav5372/gpu-io/assets/144440737/b6f962eb-dc40-4d55-a563-27ed233df146

### Particle + Grid Simulations: simulation of the network effects of a multicellular slime mold (Physarum polycephalum), based on a 2010 paper by Jeff Jones: "Characteristics of pattern formation and evolution in approximations of Physarum transport networks" and work by Sage Jenson. In this simulation, simple particle "agents" interact with an underlying field of chemical attractants to form complex, emergent behaviors.

### Experience the Physarum Transport network simulation's dynamics in this illuminating video presentation :

#### Part-1: Presenting Net, Dots, Honeycomb, Fingerprints, and Fibers simulation.

https://github.com/Malav5372/gpu-io/assets/144440737/7cf77865-582b-4682-938b-ee7b2b98b5c0

#### Part-2: watch fibers in action 
   
https://github.com/Malav5372/gpu-io/assets/144440737/96d0695e-9203-4e5a-9b1f-38901fe8820b


### 2D Wave Equation Simulation: simulates 2D wave equation and renders the resulting height field as a 3D mesh Additionally, it simulates realtime caustics formed by light shining through the 3D surface and renders the result as a texture (method described in more detail by Evan Wallace). All computations are performed in gpu-io and the results are passed to Threejs without ever leaving the GPU.

### 2D Wave Equation Simulation In Action:

https://github.com/Malav5372/gpu-io/assets/144440737/ec993fdd-e5db-4b6b-972f-a0592a684ac3


### Conway's Game of Life: The Game of Life, also known simply as Life, is a cellular automaton devised by the British mathematician John Horton Conway in 1970.It is a zero-player game, meaning that its evolution is determined by its initial state, requiring no further input. One interacts with the Game of Life by creating an initial configuration and observing how it evolves. It is Turing complete and can simulate a universal constructor or any other Turing machine.

### watch Conway's game of life in action:


https://github.com/Malav5372/gpu-io/assets/144440737/6fa26e4c-d6c1-49ad-b3d3-7194169da32c


### Julia Set Fractal:In the context of complex dynamics, a branch of mathematics, the Julia set and the Fatou set are two complementary sets (Julia "laces" and Fatou "dusts") defined from a function. Informally, the Fatou set of the function consists of values with the property that all nearby values behave similarly under repeated iteration of the function, and the Julia set consists of values such that an arbitrarily small perturbation can cause drastic changes in the sequence of iterated function values. Thus the behavior of the function on the Fatou set is "regular", while on the Julia set its behavior is "chaotic".


https://github.com/Malav5372/gpu-io/assets/144440737/1744422a-e428-4399-b88c-9b7ea6bf6ac9


### Import into HTML

*OR* you can add gpu-io.js or gpu-io.min.js to your html directly:

```html
<html>
  <head>
    <script src="gpu-io.js"></script>
  </head>
  <body>
  </body>
</html>
```

GPUIO will be accessible globally:

```js
const { GPUComposer, GPULayer, GPUProgram } = GPUIO;
```


### Use

A simple example of how to use gpu-io to simulate 2D diffusion:

```js
import {
  GPUComposer,
  GPULayer,
  GPUProgram,
  renderAmplitudeProgram,
  FLOAT,
  INT,
  REPEAT,
  NEAREST,
} from 'gpu-io';

// Init a canvas element.
const canvas = document.createElement('canvas');
document.body.appendChild(canvas);

// Init a composer.
const composer = new GPUComposer({ canvas });

// Init a layer of float data filled with noise.
const noise = new Float32Array(canvas.width * canvas.height);
noise.forEach((el, i) => noise[i] = Math.random());
const state = new GPULayer(composer, {
  name: 'state',
  dimensions: [canvas.width, canvas.height],
  numComponents: 1, // Scalar state has one component.
  type: FLOAT,
  filter: NEAREST,
  // Use 2 buffers so we can toggle read/write
  // from one to the other.
  numBuffers: 2,
  wrapX: REPEAT,
  wrapY: REPEAT,
  array: noise,
});

// Init a program to diffuse state.
const diffuseProgram = new GPUProgram(composer, {
  name: 'render',
  fragmentShader: `
    in vec2 v_uv;

    uniform sampler2D u_state;
    uniform vec2 u_pxSize;

    out float out_result;

    void main() {
      // Compute the discrete Laplacian.
      // https://en.wikipedia.org/wiki/Discrete_Laplace_operator
      float center = texture(u_state, v_uv).x;
      float n = texture(u_state, v_uv + vec2(0, u_pxSize.y)).x;
      float s = texture(u_state, v_uv - vec2(0, u_pxSize.y)).x;
      float e = texture(u_state, v_uv + vec2(u_pxSize.x, 0)).x;
      float w = texture(u_state, v_uv - vec2(u_pxSize.x, 0)).x;
      const float diffusionRate = 0.1;
      out_result =
        center + diffusionRate * (n + s + e + w - 4.0 * center);
    }
  `,
  uniforms: [
    { // Index of sampler2D uniform to assign to value "u_state".
      name: 'u_state',
      value: 0,
      type: INT,
    },
    { // Calculate the size of a 1 px step in UV coordinates.
      name: 'u_pxSize',
      value: [1 / canvas.width, 1 / canvas.height],
      type: FLOAT,
    },
  ],
});

// Init a program to render state to canvas.
// for more built-in GPUPrograms to use.
const renderProgram = renderAmplitudeProgram(composer, {
  name: 'render',
  type: state.type,
  components: 'x',
});

// Simulation/render loop.
function loop() {
  window.requestAnimationFrame(loop);

  // Diffuse state and write result to state.
  composer.step({
    program: diffuseProgram,
    input: state,
    output: state,
  });

  // If no "output", will draw to canvas.
  composer.step({
    program: renderProgram,
    input: state,
  });
}
loop(); // Start animation loop.
```

b


### Examples

Check out the Examples page to really understand how gpu-io works and how to easily create touch interactions in your application.
Source code for all examples can be found in the examples folder.

Please let me know if you made something with gpu-io!  Feel free to also post a link in the Show and Tell discussions thread.  I'll periodically add some of these to the Examples page as well.


### API

Full API documentation can be found in the docs folder.

More information about writing GLSL shaders for gpu-io can be found at docs/GLSL


### Compatibility with Threejs

gpu-io can share a webgl context with Threejs so that both libraries will be able to access shared memory on the gpu:

```js
import THREE from 'three';
import {
  GPUComposer,
  GPULayer,
  FLOAT,
  CLAMP_TO_EDGE,
  LINEAR,
} from 'gpu-io';

const renderer = new THREE.WebGLRenderer();
// Use renderer.autoClear = false if you want to overlay threejs stuff
// on top of things rendered to the screen from gpu-io.
// renderer.autoClear = false;

const composer = GPUComposer.initWithThreeRenderer(renderer);
```

Data is passed between gpu-io and Threejs via WebGLTextures.  To bind a GPULayer to a Threejs Texture:

```js
const layer1 = new GPULayer(composer, {
  name: 'layer1',
  dimensions: [100, 100],
  type: FLOAT,
  numComponents: 1,
});

const texture = new THREE.Texture();
// Link webgl texture to threejs object.
layer1.attachToThreeTexture(texture);

// Use texture in threejs scene.
const mesh = new THREE.Mesh(
  new PlaneBufferGeometry(1, 1),
  new MeshBasicMaterial({
    map: texture,
  }),
);

loop() {
  // Undo any changes threejs has made to global WebGL state.
  composer.undoThreeState();

  // Compute things with gpu-io.
  composer.step({
    program: myProgram,
    output: layer1,
  });

  ....

  // Reset global WebGL state back to what threejs is expecting
  // (otherwise we get WebGL errors).
  composer.resetThreeState();
  // Render threejs scene.
  // Updates to layer1 will propagate to texture without any
  // additional needsUpdate flags.
  renderer.render(scene, camera);
}
```

More info about using gpu-io with Threejs can be found in the Threejs Example


### Limitations/Notes


### Limitations

- gpu-io currently only supports GPULayers with 1D or 2D arrays of dense data.  3D textures are not officially supported by the library e.g. there are not currently 3D GPULayers, though I am looking at adding this.  If you want to compute 3D simulations on triangle meshes in gpu-io, you will just need to pass in your 3D position data as a 1D list to a GPULayer and then access it in the fragment shader using .xyz.  TODO: make example for this.
- gpu-io does not currently allow you to pass in your own vertex shaders.  Currently all computation is happening in user-specified fragment shaders; vertex shaders are managed internally.
- In order for the WRAP/FILTER polyfilling to work correctly, any calls to texture() must contain a direct reference to the sampler2D that it should operate on.  For example:

```glsl
varying vec2 v_uv;

uniform sampler2D u_sampler1;
uniform sampler2D u_sampler2;

out vec4 out_result;

vec4 lookupSampler2(vec2 uv) {
  // This is good, it passes u_sampler2 directly to texture().
  return texture(u_sampler2, uv);
}

vec4 lookupSampler(sampler2D sampler, vec2 uv) {
  // At compile time it is hard to say which sampler
  // is passed to texture().
  // This will not be polyfilled, it will throw a warning.
  return texture(sampler, uv);
}

void main() {
  // This is good, it passes u_sampler1 directly to texture().
  vec2 position = texture(u_sampler1, v_uv).xy;

  ....
}
```

### GLSL Version

gpu-io defaults to using WebGL2 (if available) with GLSL version 300 (GLSL3) but you can set it to use WebGL1 or GLSL version 100 (GLSL1) by passing `contextID` or `glslVersion` parameters to `GPUComposer`:

```js
import {
  GPUComposer,
  GLSL1,
  WEBGL1,
} from 'gpu-io';

// Init with WebGL2 (if available) with GLSL1.
const composer1 = new GPUComposer({
  canvas: document.createElement('canvas'),
  glslVersion: GLSL1,
});

// Init with WebGL1 with GLSL1 (GLSL3 is not supported in WebGL1).
const composer2 = new GPUComposer({
  canvas: document.createElement('canvas'),
  contextID: WEBGL1,
});
```

See docs>GPUComposer>constructor for more information.

gpu-io will automatically convert any GLSL3 shaders to GLSL1 when targeting WebGL1.  If supporting WebGL1/GLSL1 is important to you, see the GLSL1 Support doc for more info about what functions/types/operators are available in gpu-io's flavor of GLSL1.


### Acknowledgements

I used a few codebases as reference when writing this, thanks to their authors for making these repos available:

- [three.js](https://github.com/mrdoob/three.js/)
- [regl](https://github.com/regl-project/regl)
- [gpu.js](https://github.com/gpujs/gpu.js/)
- [WebGL Boilerplate](https://webglfundamentals.org/webgl/lessons/webgl-boilerplate.html)
- [GPU Accelerated Particles with WebGL 2](https://gpfault.net/posts/webgl2-particles.txt.html)

Other resources:

- [WebGL best practices](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API/WebGL_best_practices) by Mozilla
- [Compatibility issues in Shadertoy / webGLSL](https://shadertoyunofficial.wordpress.com/2016/07/22/compatibility-issues-in-shadertoy-webglsl/) Shadertoy – Unofficial

