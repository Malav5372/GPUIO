### GPUIO

GPUIO is a project in the domain of computer graphics and GPU computing, specializing in physics simulations. It harnesses the capabilities of modern GPUs, supporting WebGL 1/2 and GLSL 1/3, and offers the 2 simulation types:

1. Conway's Game of Life: Demonstrates emergent behavior through cellular automata.

2. Julia Set Fractal: Creates intricate fractal art using complex mathematics.

- [Use](#use)
- [Compatibility with Threejs](#compatibility-with-threejs)
- [Acknowledgements](#acknowledgements)


### Conway's Game of Life: The Game of Life, also known simply as Life, is a cellular automaton devised by the British mathematician John Horton Conway in 1970.It is a zero-player game, meaning that its evolution is determined by its initial state, requiring no further input. One interacts with the Game of Life by creating an initial configuration and observing how it evolves. It is Turing complete and can simulate a universal constructor or any other Turing machine.

### watch Conway's game of life in action:


 
### Julia Set Fractal:In the context of complex dynamics, a branch of mathematics, the Julia set and the Fatou set are two complementary sets (Julia "laces" and Fatou "dusts") defined from a function. Informally, the Fatou set of the function consists of values with the property that all nearby values behave similarly under repeated iteration of the function, and the Julia set consists of values such that an arbitrarily small perturbation can cause drastic changes in the sequence of iterated function values. Thus the behavior of the function on the Fatou set is "regular", while on the Julia set its behavior is "chaotic".

### watch it in action:


### Compatibility with Threejs

GPUIO can share a webgl context with Threejs so that both libraries will be able to access shared memory on the gpu:

```js
import THREE from 'three';
import {
  GPUComposer,
  GPULayer,
  FLOAT,
  CLAMP_TO_EDGE,
  LINEAR,
} from 'GPUIO';

const renderer = new THREE.WebGLRenderer();
// Use renderer.autoClear = false if you want to overlay threejs stuff
// on top of things rendered to the screen from GPUIO.
// renderer.autoClear = false;

const composer = GPUComposer.initWithThreeRenderer(renderer);
```

Data is passed between GPUIO and Threejs via WebGLTextures.  To bind a GPULayer to a Threejs Texture:

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

  // Compute things with GPUIO.
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

### GLSL Version

GPUIO defaults to using WebGL2 (if available) with GLSL version 300 (GLSL3) but you can set it to use WebGL1 or GLSL version 100 (GLSL1) by passing `contextID` or `glslVersion` parameters to `GPUComposer`:

```js
import {
  GPUComposer,
  GLSL1,
  WEBGL1,
} from 'GPUIO';

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

### Acknowledgements

we used a few codebases as reference when writing this, thanks to their authors for making these repos available:

- [three.js](https://github.com/mrdoob/three.js/)
- [regl](https://github.com/regl-project/regl)
- [gpu.js](https://github.com/gpujs/gpu.js/)
- [WebGL Boilerplate](https://webglfundamentals.org/webgl/lessons/webgl-boilerplate.html)
- [GPU Accelerated Particles with WebGL 2](https://gpfault.net/posts/webgl2-particles.txt.html)

Other resources:

- [WebGL best practices](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API/WebGL_best_practices) by Mozilla
- [Compatibility issues in Shadertoy / webGLSL](https://shadertoyunofficial.wordpress.com/2016/07/22/compatibility-issues-in-shadertoy-webglsl/) Shadertoy – Unofficial

