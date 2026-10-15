# SHADER Essay
# Language
This is coded in 1 specific shader language: WGSL (**W**eb**G**pu **S**hader **L**anguage)
Docs for that can be found with URL below, that is its own seperate subject and even idk how it works entirely just yet
`webgpufundamentals.org/webgpu/lessons/webgpu-wgsl.html`
Those are best docs i could find, but be aware there are multiple docs so if you find better alternatives let me know
for now ill leave rough understanding here and summary from above here

Now this isnt the only thing used, bloxd also has callbacks in the sense that bloxd calls apon these for usage
Those will be listed below
but first, file structure
- Structure
```js
MyShader/
  pack.json
  someName.wgsl
  someOtherName.wgsl
  whateverYouWant.wgsl
  ...
```
- Pack.json
```js
{
    "apiVersion": 1,
    "name": "My Pack"
    "author": "optional name"
}
```
apiVersion must be 1
name can be whatever you want
author is optional
# Bloxd Shader API
## Macros
these are toggles in your code, based on how and where its used, will alter your code and or api as well
```wgsl
BLOXD_STRICT // "use strict" of shaders, prevent stuff like `discard;` from being used
BLOXD_LIGHTING // bloxd lighting toggle
BLOXD_ALPHA // is shader accepting alpha values
BLOXD_SCENE_DEPTH // not too sure
//they are either defined or not
//parse directive exists and are used like so

#ifdef MACRO
  // runs code inside here if `MACRO` is defined
#else
  // runs code inside here if `MACRO` isnt defined
#endif
// similar structure to if(...){...}else{...} in JS

//#ifndef //if NOT defined, the negation of ifdef
```
## Struct
These can be thought as types
```wgsl
struct bloxd_VertexIn {
    position: vec3f,  // noa-local world space (world minus originOffset)
    normal: vec3f,
    originOffset: vec3i, // whole blocks
    time: f32, // seconds, wrapped every 3600 s (part of the API)
}

#ifndef BLOXD_STRICT //exists only if not STRICT
struct bloxd_Vertex { 
    position: vec3f, // noa-local
}
#endif

struct bloxd_SurfaceIn {
    colour: vec3f, // texture x vertex colour (tint x AO, baked together by the mesher in v1)
    alpha: f32,
    normal: vec3f,
    position: vec3f, // noa-local
    originOffset: vec3i, // a face's world block is vec3i(floor(position - normal * 0.5)) + originOffset
    eyePosition: vec3f, // noa-local
    time: f32,
    sunDirection: vec3f, // unit, towards the sun, in world orientation; zero when the sky has no sun
}

struct bloxd_Surface {
    colour: vec3f,
#ifdef BLOXD_ALPHA
    alpha: f32, //if ALPHA is being used, allow alpha value to be passed
#endif
    emission: vec3f, // added after lighting, then fogged
}

struct bloxd_LightIn {
    engineColour: vec3f, // the engine's result for this fragment, from the surface hook's colour
    directional: f32, // summed Lambert from the scene's directional lights (0 when off)
    ambient: vec3f, // the material's ambient times the scene's
#ifdef BLOXD_LIGHTING //if LIGHTING used, allow these extra keys
    blockLight: vec3f, // voxel block light, tone-mapped
    skyLight: f32, // voxel sky light, tone-mapped
    ambientLightColour: vec3f,
    skyLightColour: vec3f,
    heldLight: vec3f, // held light here, tone-mapped like blockLight, before the ambient boost; 0 out of reach
#endif
    fogColour: vec3f,
    fogFactor: f32, // 1 = no fog, 0 = fully fogged
    position: vec3f, // as in bloxd_SurfaceIn
    normal: vec3f,
    originOffset: vec3i,
    eyePosition: vec3f,
    time: f32,
    sunDirection: vec3f,
}

struct bloxd_Light {
    colour: vec3f,
}
```
## API
you call these functions to see what bloxd would do, from there you can edit that result
```wgsl
#ifndef BLOXD_STRICT //only work if STRICT mode
// takes a vertex in
fn bloxd_defaultVertex(v: bloxd_VertexIn) -> bloxd_Vertex { //return a position
    return bloxd_Vertex(v.position);
}
#endif

//
fn bloxd_defaultSurface(s: bloxd_SurfaceIn) -> bloxd_Surface { //return color, and alpha if using
#ifdef BLOXD_ALPHA
    return bloxd_Surface(s.colour, s.alpha, vec3f(0.0));
#else
    return bloxd_Surface(s.colour, vec3f(0.0));
#endif
}

//given a surface, and engine light obj
fn bloxd_defaultLight(s: bloxd_Surface, l: bloxd_LightIn) -> bloxd_Light { //return a color
    return bloxd_Light(l.engineColour + s.emission * l.fogFactor);
}
```
## Callbacks
You return the results back into these callbacks
```wgsl
fn bloxd_terrain_vertex(v: bloxd_VertexIn) -> bloxd_Vertex {} //pos
fn bloxd_terrain_surface(v: bloxd_SurfaceIn) -> bloxd_Surface {} //color, alpha
fn bloxd_terrain_light(s: bloxd_Surface, l: bloxd_LightIn) -> bloxd_Light {} //color

fn bloxd_liquid_vertex(v: bloxd_VertexIn) -> bloxd_Vertex {} //pos
fn bloxd_liquid_surface(v: bloxd_SurfaceIn) -> bloxd_Surface {} //color, alpha
fn bloxd_liquid_light(s: bloxd_Surface, l: bloxd_LightIn) -> bloxd_Light {} //color
```
# Example
```wgsl
fn bloxd_terrain_surface(s: bloxd_SurfaceIn) -> bloxd_Surface {
    return bloxd_defaultSurface(s);
}
//shader just returns what was given in
```
```wgsl
fn bloxd_terrain_surface(s: bloxd_SurfaceIn) -> bloxd_Surface {
    var result = bloxd_defaultSurface(s);
    result.colour = vec3f(0.0) + 0.5 * result.colour;
    return result;
}
//shader which makes all darker
//Change the 0.0 or 0.5 see what happens
```
```wgsl
fn bloxd_terrain_surface(s: bloxd_SurfaceIn) -> bloxd_Surface {
    var result = bloxd_defaultSurface(s);
    let pulse = 0.5 + 0.5 * sin(s.time * 3.0); //what if its greater than 3.0 ?
    result.colour = vec3f(pulse, 0.0, 0.0);
    return result;
}
//Pulsing animation
```
- full example
```wgsl
fn bloxd_terrain_surface(s: bloxd_SurfaceIn) -> bloxd_Surface {
    var result = bloxd_defaultSurface(s);

    // Get the block's world position
    let worldPos = s.position + vec3f(s.originOffset);

    // Create a repeating pattern
    let pattern = sin(worldPos.x * 2.0 + s.time * 3.0)
                * cos(worldPos.z * 2.0 + s.time * 2.0);

    // Turn the pattern into a colour
    let wave = pattern * 0.5 + 0.5;
    result.colour = vec3f(wave, 0.2, 1.0 - wave);

    // Make the pattern glow
    result.emission = vec3f(wave * 0.5, 0.0, wave);

    return result;
}
```
# Credits
- `GlitchHunter` me: for documentating and data mining for shaders
- `_Xenon_`: for making these shaders and thus helping to pave the way for new shaders
- `Cannoli`: for reverse engineering shader activation, and needed data
- `ObiloxYT`: helped figure out how they worked and helped with shader storage investigations
# Ping List
- all members in credits
- `Ocelote` `Tridentify`  //previous interest
- `Crownix` //wanted to be pinged for findings like these
- `nervousking` `SUSSYPRIME` //seen it be developed
- `KingCobra` //wanted to see shaders working

<details>
  
<summary>
  
## Reference

</summary>

### GLSL String found
```wgsl
// Engine-provided and read-only to packs. apiVersion 1.
struct bloxd_VertexIn {
    position: vec3f,     // noa-local world space (world minus originOffset)
    normal: vec3f,
    originOffset: vec3i, // whole blocks
    time: f32,           // seconds, wrapped every 3600 s (part of the API)
}

#ifndef BLOXD_STRICT
struct bloxd_Vertex {
    position: vec3f,     // noa-local
}
#endif

struct bloxd_SurfaceIn {
    colour: vec3f,       // texture x vertex colour (tint x AO, baked together by the mesher in v1)
    alpha: f32,
    normal: vec3f,
    position: vec3f,     // noa-local
    originOffset: vec3i, // a face's world block is vec3i(floor(position - normal * 0.5)) + originOffset
    eyePosition: vec3f,  // noa-local
    time: f32,
    sunDirection: vec3f, // unit, towards the sun, in world orientation; zero when the sky has no sun
}

struct bloxd_Surface {
    colour: vec3f,
#ifdef BLOXD_ALPHA
    alpha: f32,
#endif
    emission: vec3f,     // added after lighting, then fogged
}

struct bloxd_LightIn {
    engineColour: vec3f, // the engine's result for this fragment, from the surface hook's colour
    directional: f32,    // summed Lambert from the scene's directional lights (0 when off)
    ambient: vec3f,      // the material's ambient times the scene's
#ifdef BLOXD_LIGHTING
    blockLight: vec3f,   // voxel block light, tone-mapped
    skyLight: f32,       // voxel sky light, tone-mapped
    ambientLightColour: vec3f,
    skyLightColour: vec3f,
    heldLight: vec3f,    // held light here, tone-mapped like blockLight, before the ambient boost; 0 out of reach
#endif
    fogColour: vec3f,
    fogFactor: f32,      // 1 = no fog, 0 = fully fogged
    position: vec3f,     // as in bloxd_SurfaceIn
    normal: vec3f,
    originOffset: vec3i,
    eyePosition: vec3f,
    time: f32,
    sunDirection: vec3f,
}

struct bloxd_Light {
    colour: vec3f,
}

#ifndef BLOXD_STRICT
fn bloxd_defaultVertex(v: bloxd_VertexIn) -> bloxd_Vertex {
    return bloxd_Vertex(v.position);
}
#endif

fn bloxd_defaultSurface(s: bloxd_SurfaceIn) -> bloxd_Surface {
#ifdef BLOXD_ALPHA
    return bloxd_Surface(s.colour, s.alpha, vec3f(0.0));
#else
    return bloxd_Surface(s.colour, vec3f(0.0));
#endif
}

fn bloxd_defaultLight(s: bloxd_Surface, l: bloxd_LightIn) -> bloxd_Light {
    return bloxd_Light(l.engineColour + s.emission * l.fogFactor);
}
```

### Source Code
```js
const iy = {
    terrain: {
        vertex: "bloxd_terrain_vertex",
        surface: "bloxd_terrain_surface",
        GR: "bloxd_terrain_light"
    },
    liquid: {
        vertex: "bloxd_liquid_vertex",
        surface: "bloxd_liquid_surface",
        GR: "bloxd_liquid_light"
    }
}
  , Vs = ["vertex", "surface", "GR"]
  , os = new Set([].concat(...Object.values(iy).map((ss => Object.values(ss)))));
function Xs(ss) {
    for (const ys of Object.values(iy))
        for (const vs of Vs)
            if (ys[vs] === ss)
                return vs;
    return null
}
const Ws = {
    vertex: {
        params: [{
            name: "v",
            type: "bloxd_VertexIn"
        }],
        returns: "bloxd_Vertex"
    },
    surface: {
        params: [{
            name: "s",
            type: "bloxd_SurfaceIn"
        }],
        returns: "bloxd_Surface"
    },
    GR: {
        params: [{
            name: "s",
            type: "bloxd_Surface"
        }, {
            name: "l",
            type: "bloxd_LightIn"
        }],
        returns: "bloxd_Light"
    }
};
```

</details>

<details>
  
<summary>

## Rough Tutorial of WGSL

</summary>

---

Expected Knowledge:
- Standard JS knowledge
- A Understanding of Types

### JS vs WGSL

```js
var a = 1;
let c = 3;
function d(e) { return e * 2; }
```

```wgsl
var a: f32 = 1;
let c: f32 = 3;
fn d(e: f32) -> f32 { return e * 2; }
//note how you need variable, argument, and return types stated
```

## Syntax

```wgsl
// For
for (var i = 0; i < 10; i++) { ... }

// If
if (i < 5) {
  ...
} else if (i > 7) {
  ..
} else {
  ...
}

// While
var j = 0;
while (j < 5) {...}

// Loop
var k = 0;
loop {
  k++;
  if (...) {
    break; // Break
  }
  break if (...); // Break if
  
}

continue
for (var i = 0; i < 10; ++i) {
  if (i % 2 == 1) {
    continue; // Continue
  }
  ...
  continuing { // Continuing
    // continue goes here
    ...
  }
}

if (v < 0.5) {
  discard; // Discard
}
// discard exits the shader. It can only be used in a fragment shader.

// Switch
var a : i32;
let x : i32 = generateValue();
switch x {
  case 0: {      // The colon is optional
    a = 1;
  }
  default {      // The default need not appear last
    a = 2;
  }
  case 1, 2, {   // Multiple selector values can be used
    a = 3;
  }
  case 3, {      // The trailing comma is optional
    a = 4;
  }
  case 4 {
    a = 5;
  }
}
// switch only works with u32 or i32 and cases must be constants.
```
so unlike JS, WGSL is a strongly typed language, so it needs to know the type of all things, however there are good shortcuts for this

### Inference
so instead of having to declare all your types, it can infer your type
- Simple
```wgsl
fn f() -> bool { return false; }

// 32 Bit
1i   // i32
1u   // u32
1f   // f32
1.0f // f32

// 16 Bit
1h   // f16
1.0h // f16

// Compile Type
1    // AbstractInt
1.0  // AbstractFloat

// Inference
f() // inferred as bool
```
- Vector
```wgsl
let a = vec4<f32>(1, 2, 3, 4);
let b = vec4f(1, 2, 3, 4);
// both are same type
```

### Types
- Simple Types
```wgsl
i32  // 32 bit signed int
u32  // 32 bit unsigned int
f32  // 32 bit float
bool // boolean

// COMPILE ONLY
AbstractInt // undecided int type
AbstractFloat // undecided float type
```
- Read/Write
```wgsl
var a = 1 // is equivalent of `let` in JS
let b = 1 // is equivalent of `const` in JS

const c = 1 // is a compile time constant
```
- Vectors
```wgsl
// Declare
let a = vec4<f32>(1, 2, 3, 4); // 4 f32 s
let b = vec4<f32>(4, 3, 2, 1); // 4 f32 s

let uint = vec3<u32>(3.4, 5.6, 7.8); // 3 u32 s
let sint = vec2<i32>(1, -2); // 2 i32 s

// Accessing
a.z;   // via x,y,z,w
a.b;   // via r,g,b,a
a[2];  // via array element accessors

// Swizzles
a.zx;   // via x,y,z,w
a.br;   // via r,g,b,a
vec2<f32>(a[2], a[0]);
a.xzz // even repeat elements

// Insertion
vec4f(1, 2, 3, 4);
vec2f(2, 3);
vec4f(1, a.xx, 4);
vec4f(1, a.yz, 4);
vec4f(a.xyz, 4);
vec4f(1, a.yzw);
// all the same type
// it just inserts the vector into the longer vector

// Operations
a + b; // vec4f(5, 5, 5, 5)
a * b; // vec4f(4, 6, 6, 4)
a - b; // vec4f(-3, -1, 1, 3)
mix(a, b) // vec4f(1, 2, 2, 1)
```
- Matrix
```wgsl
// `mat[numVectors]x[vectorSize]<[type]>`
let a: mat4x4<f32> = ...
let b: mat4x4f = ...
let c = mat4x4f(...);
let v = vec4f(1, 2, 3, 4);

a[2];   // vec4f of the 3rd vector of a
a * v;  // vec4f and the result of a * b ( * being used as dot product )
```
- Array
```wgsl
// `array<[type],[numElements]>`
let a = array<f32, 5>;   // an array of five f32s
let b = array(vec3f(1,2,3), vec3f(4,5,6), vec3f(7,8,9));
```
- Function
```wgsl
// `fn [name]([params]) -> [returnType] {[body]}`
fn add(a: f32, b: f32) -> f32 {
  return a + b;
}
```
## Conversion
Since it is a strongly typed language you cant auto convert one to another
```wgsl
let a = 1;     // a is an i32
let b = 2.0;   // b is a f32
let c = a + b; // ERROR can't add an i32 to an f32
```
and instead must do
```wgsl
let a = 1; // a is an i32
let b = 2.0; // b is a f32
let c = f32(a) + b; // ok
let d = 1 + 2.0; // d is a f32 // AbstInt + AbstFloat = AbstFloat = f32 //then becomes decided
```

Now imma leave the rest to
- https://webgpufundamentals.org/webgpu/lessons/webgpu-wgsl.html
- https://webgpufundamentals.org/webgpu/lessons/webgpu-wgsl-function-reference.html
But that has been a rough tutorial into WGSL
---

</details>

