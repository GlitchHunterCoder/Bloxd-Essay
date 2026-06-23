# ESSAY v2

## **Bloxd.io Engine & Execution Model** 

A comprehensive technical essay on code execution, interruption safety, and engine internals 

## **GlitchHunterCoder** 

As of 27 May 2026 

**BLOXD.IO ENGINE & EXECUTION MODEL** 

GlitchHunterCoder · 2026 

## **Table of Contents** 

- **1. Globals** 
  - 1.1 Static Globals
  - 1.2 Injected Globals 
- **2. Flow of Execution** 
  - 2.1 Error Stacks & Code Wrapping 
  - 2.2 Variable Scope & the Escape Trick 
- **3. Execution Steps** 
  - 3.1 Step_Code()
  - 3.2 ByteCode
  - 3.3 Code Blocks 
  - 3.4 Chaining 
- **4. World Code** 
  - 4.1 Callback Extraction
  - 4.2 Event Loop 
- **5. Limitations** 
- **6. History of Anti-Interrupt** 
  - Era 1 Anywhere Interrupt — Idempotence & Atomicity
  - Era 2 State Machines
  - Era 3 Cost Reduction & Batch Testing
  - Era 4 InternalError Hook
  - Era 5 Generators
  - Era 6 eval() as Voluntary Interrupt Point
  - Era 7 api.isNearInterrupt() 
- **7. Rate, Text & Item Limits** 
  - 7.1 Rate Limit 
  - 7.2 Text Limit 
  - 7.3 Item Limit 
- **8. Outro**

## **1. Globals** 

## **1.1 Static Globals** 

Bloxd exposes a fixed set of global identifiers inside the QuickJS sandbox, organised into the following categories: 
- **Natives** — Object, Function, Array, Number, Boolean, String, Symbol, RegExp 
- **Simple fns** — parseInt, parseFloat, isNaN, isFinite 
- **Escape** — escape, unescape 
- **ArrayBuffer** — ArrayBuffer, SharedArrayBuffer, DataView 
- **Error types** — Error, EvalError, RangeError, ReferenceError, SyntaxError, TypeError, URIError, InternalError, AggregateError 
- **URI fns** — decodeURI, decodeURIComponent, encodeURI, encodeURIComponent
- **Primitives** — Infinity, NaN, undefined 
- **Maps / Sets** — Map, Set, WeakMap, WeakSet 
- **Typed Arrays** — Uint8ClampedArray … Float64Array (10 types) 
- **Injected** — api, console, Date, myId, playerId, thisPos
- **Introspective** — Reflect, Proxy
- **Misc** — Math, eval, globalThis, JSON

Two accessible intrinsics also exist: `GeneratorFunction` and `Generator` . Async functions can be declared but cannot be called — they throw _Error: Not a Function_ and have no prototype. 

## **Notable globalThis quirks** 
- Most exposed methods are long-deprecated (e.g. `String.prototype.anchor` , `RegExp.prototype.compile` ). 
- Some properties exist but are inaccessible: `Function.prototype.fileName` , `Symbol.asyncIterator` . 
- Some have no specification: `Symbol.operatorSet` , `Object.__getClass` . 
- All api function `.length` values are always 0. 
## **1.2 Injected Globals** 

A second group is dynamically injected on _every_ code execution — discovered by proxying `globalThis` and observing GET/SET operations: 

_Fig 2 — Proxied globalThis access log showing SET/GET operations on injected globals_ 
```C
SET: api = {}
SET: console = {}
SET: Date = {}
SET: myId = -1
SET: playerId = -1
SET: thisPos = [0,0,0]
GET: api
GET: console
GET: Date
```

|**Operation**|**Keys**|**Presence**|**Description**|
|---|---|---|---|
|SET ?|...allCallbacks|all or nothing|Sets engine-side callbacks|
|SET|api, console, Date|always|Sets necessary values|
|SET|myId ?, playerId ?,<br>thisPos ?|each independently optional|Sets code-block specifics|
|GET|api, console, Date|always — Date signals boot<br>end|Checks necessary values|
- The injected set is re-applied on _every_ execution, not just at startup. 
- Callbacks are registered when the engine GETs each callback name after the first world-code run. 
- The final GET of api/console/Date is a health check — broken invariants trigger _`Error: Please Report to Developers on Discord`_ . 
## **2. Flow of Execution** 

_Fig 3 — Full execution model map (CodeBlock · RateLimit_Check · Interruption_Check · Step_Code · WorldCode)_ 
<img width="1097" height="1079" alt="image" src="https://github.com/user-attachments/assets/7113f841-5c5c-4bdf-8cf1-25ddca64d72e" />

## **2.1 Error Stacks & Code Wrapping** 

By intentionally triggering errors and inspecting their stacks, the internal wrapping structure can be inferred. Key observations: 
- Syntax errors report `eval.js:1` . 
- Constructed errors ( `throw new Error()` ) report `<eval>` . 
- `new Function()` errors appear at line 3 — implying two header lines exist before user code. 

Inferred wrapper structure: 

```js
// line 1  (hidden)
// line 2  (hidden)
eval(`{${userCode}}`)   // line 3 — where errors originate
```

## **2.2 Variable Scope & the Escape Trick** 

Because user code lives inside a block scope, deliberately closing that block early lets variables escape to the outer (global) scope: 

```js
  let varLocal = "block-scoped"
};                              // escape the block
let varGlobal = "globally scoped via escape trick"
{                               // re-enter block
  console.log(varGlobal)        //  works
  console.log(varLocal)         //  ReferenceError
```

`var` declarations and function declarations also escape and persist across separate code-block executions — enabling global-local variables without touching `globalThis` . 

## **Hoisting and Scoping rules** 

variable hoisting is shown via the key below
- `(Name)` — where the variables name now exists (`Declaration hoisting`)
- `(Value)` — where the variables value now exists (`Value hoisting`)
- `(End)` — from where the variables no longer exists
now if we look at this format the pattern becomes apparant quickly

- `KEYWORD: let / const`
  - `SCOPE: blocks`
  - `HOIST: name only + (TDZ)`
```js
// console.log(a) // throws `ReferenceError`

;{

  // console.log(a) // throws `ReferenceError`
  let a = "LET" // (Name) // (Value)
  console.log(a) // logs: "LET"
  
  // (End)
}; 

// console.log(a) // throws `ReferenceError`
```
- `KEYWORD: var`
  - `SCOPE: function body`
  - `HOIST: name only (value is "undefined")`
```js
// (Name)
console.log(a) // logs: undefined 

;{  // (Value) = `undefined`

  console.log(a) // logs: undefined
  var a = "VAR" // (Value)
  console.log(a) // logs: "VAR"

};

console.log(a) // logs: "VAR"
// (End)
```
- `KEYWORD: function`
  - `SCOPE: function body`
  - `HOIST: name and value (value hoisted to top of Block Scope)`
```js
// (Name)
console.log(a) // logs: undefined

;{ // (Value)

  console.log(a) // logs: function
  function a(){}
  console.log(a) // logs: function

};

console.log(a) // logs: function
// (End)
```

## **3. Execution Steps** 

## **3.1 Step_Code()** 
- **1 Rate Limit Check** — If it fails the run is aborted immediately. 
- **2 Reset IU** — Interruption Unit counter is set back to 0. 
- **3 ByteCode loop** — Execute instructions one at a time. 
- **4 Completion check** — If done, end; otherwise run Interruption_Check() and loop. 
## **3.2 ByteCode** 
- **One instruction** from compiled QuickJS bytecode is executed per iteration. 
- **IU update:** whitelisted opcodes contribute 0 IU; all others increment by 1. (The whitelist was lost during the eval patch, temporarily breaking IU tracking.) 
- **RT update:** runtime is accumulated per instruction or recalculated at each check point. 
## **3.3 Code Blocks** 
- **Initialisation:** injects api, console, Date, myId, playerId, thisPos before execution. 
- **Return value:** after execution the return is `JSON.stringify` -ed and emitted. JSON.stringify is a core engine theme — it governs text limits and API sanitisation. 
## **3.4 Chaining** 
Blocks can chain — a recursive, stack-based activation of neighbouring blocks (Comfirmed by Slushie): 
```js
done  = []   // already executed
stack = []   // queued
function Chain(pos) {
  try {
    let output = exec(pos)
    let [x,y,z] = pos
    let adj = [
      [x+1,y,z], [x-1,y,z],
      [x,y+1,z], [x,y-1,z],
      [x,y,z+1], [x,y,z-1],
    ]
    adj.forEach(item => {
      if (!stack.includes(item) && !done.includes(item))
        stack.push(item)
    })
    sendReturn(output)
  } catch(err) {
    sendError(err)
  } finally {
    let next = stack.pop()
    done.push(next)
    if (next !== undefined) Chain(next)
  }
}
```

Chaining enables: 
- **Chains** — all blocks must succeed. 
- **Branching** — blocks react to prior success/failure. 
- **Branch slicing** — execute a later block directly, skipping earlier ones. 
- **Interruption handling** — outer blocks loop infinitely and interrupt, but branching lets the centre block complete regardless. 
## **4. World Code** 
- No chaining — only one world code instance exists. 
- No displayed return value — replaced by callback registration result. 
- The return-too-large limit still applies. 
## **WorldCode sequence** 
1. Run first pass via Step_Code(). 
2. GET all callback names; check each is undefined or a function. 
3. LOOP: gather events → call each callback once per event → wait 50 ms → repeat. 
## **4.1 Callback Extraction** 

_Fig 4 — Available callback names as seen by the engine_ 
```js
GET: tick
GET: onClose
GET: onPlayerJoin
GET: onPlayerLeave
GET: onPlayerJump
GET: onRespawnRequest
GET: playerCommand
GET: onPlayerChat
GET: onPlayerChangeBlock
GET: onPlayerDropItem
```

After the first run the engine GETs every callback name. Non-functions are logged ( _<Name> is assigned to a non-function_ ) without throwing. Valid callbacks are **copied by object reference** — reassigning the variable later has no effect: 

```js
// World code
let Ref = () => console.log("old") tick = () => Ref()      // engine copies the wrapper function reference
```
```js
// Code block (later) Ref = () => console.log("new")   // effective — wrapper calls the new Ref
```

_**Discovery:** a defineProperty getter returning a function is fully respected by the extractor._ 

## **4.2 Event Loop** 
- `Date.now()` is cached once per tick cycle, not per callback. 
- Callback execution order is non-deterministic within a tick (confirmed by Tom). 
- Code-block execution order within the loop is also random. 

_Fig 5 — Ordering test: onPlayerClick vs onPlayerClickUp order is non-deterministic_ 
```js
Log: ["onBlockStand","tick","onPlayerClick","onPlayerClickUp"]
Log: ["onBlockStand","tick","onPlayerClickUp","onPlayerClick"]
```

## **5. Limitations** 

|**Category**|**Detail**|
|---|---|
|Interruption|5,000 IU + Runtime threshold (TU)|
|Rate Limit|Code/Board/eval: N ms per X s window (up to 7 s); Chunk loading: up to 1 min; Block data;<br>Code collab|
|Text Limit|WC/CB character caps; Schematic/world chunk caps; 2 KB: fn args, item attrs; 10 KB: lobby<br>db, player db|
|Item Limit|200 meshes (no physics); 50 meshes (with physics); Particles; Mobs|

## **5.1 Interruption Limit — Detail** 

The condition built into QuickJS that Bloxd registers: 

```js
if (IU % 5000 == 0 && TU > MAX_RUNTIME_ALLOWED) {
  interrupt()
}
```

**IU (Interruption Units)** — Increments per non-whitelisted bytecode instruction. 

**TU (Temporal Units)** — Time derived from code execution — not Date.now() (which is cached). 

## **6. History of Anti-Interrupt** 

## **Terminology** 
- **Idempotent** — Running a function multiple times = same result as once. 
- **Atomic** — Either fails with zero side-effects, or completes fully. 
- **Weak Interruption Safety** — Interrupt isolates to current op; future ticks unaffected. (Atomic, not idempotent.) 
- **Strong Interruption Safety** — Can recover current execution after interrupt AND protect future ticks. (Atomic + idempotent.) 
those are not the only definitions, less common terms have arrived such as
- **RICO-INT** — recallable if cut off by interruption, example below
```js
// suppose some variable is defined in a way tied to the function
let some_var = false;

let array1=[];
let array2=[];

let func = function() {
    if(!some_var) array1.push(1);
    some_var = true;
    array2.push(2);
}
```
now below

Reference code used across all eras: 

```js
let inx = 3
while (inx < 1000) {
  let a = true
  for (let i = 3; i < inx; i += 2) {
    if (inx % i == 0) { a = false }
  }
  if (a) { console.log(inx) }
  inx += 2
}
```

## **Era 1 — Anywhere Interrupt — Idempotence & Atomicity** 

The community assumed interrupts could occur anywhere. Solution: never advance state until all required work for that step was 100% complete. 

```js
let inx=null, i=null, a=null, start=0, cmd=null
function tick(){
  if(cmd==null){ start=0 }
  if(cmd=="prime"){
    if(start==0){ [inx,i,a]=[3,3,true]; start=1; }
    if(inx>=1000){ cmd=null; return }
    if(start==1){
      if(i>=inx){ start=2; return }
      if(inx%i==0){ a=false }; i+=2; return
    }
    if(a){ console.log(inx) }
    [a,i,inx]=[true,3,inx+2]; start=1
  }
}
if(start==0){ cmd="prime" }
```

## **Era 2 — State Machines** 

Define discrete atomic states each safely interruptible, with a clear transition map. Remains popular to this day for its versatility. 

```js
let state=1, inx=null, i=null, a=null
const Atoms = [
  ()=>{},                                 // 0: Null
  
  ()=>{ [inx,i,a]=[3,3,true]; state=2 },  // 1: Init
  
  ()=>{                                   // 2: Inner loop
    if(inx>=1000){state=0;return}
    if(i>=inx){state=3;return}
    if(inx%i==0){a=false}; i+=2
  },
  
  ()=>{                                   // 3: Outer loop
    if(a){console.log(inx)}
    [a,i,inx]=[true,3,inx+2]; state=2
  }
]
function tick(){ Atoms[state]() }
```

## **Era 3 — Cost Reduction & Batch Testing** 

Sulfrox's interruption tester enabled quantitative IU measurement. A 3-day session in BCOP server (GlitchHunterCoder, delfineonx, chmod, the_ccccc, dulph, sulfrox — 1,000+ messages) produced these IU cost rules: 
- Bitwise ops + element access < if-statements in IU cost. 
- String concatenation cheaper than template literals. 
- Assignment / deletion / variable changes cost 0 IU. 
- While loops cheaper than for loops. 
- IU increments on branches (if, while condition, for, etc). 
- new expression = 1 IU. Function call = 1 IU. 
and a strong intuition for measuring and finding when IU could be used

```js
// Optimised prime sieve
let inx = 3
while (inx < 1000) {
  let a = true, i = 3
  while (i < inx) {
    a = !!(+a & +(inx%i == 0))   // bitwise instead of if
    i += 2
  }
  [()=>{}, console.log][+a](inx) // array lookup instead of if
  inx += 2
}
```

## **IU vs TU debate** 

No consensus was reached: optimise for fewer IU increments (avoid hitting 5,000) or lower runtime (TU)? The answer depends on whether you aim to prevent interrupts entirely, or just stop them from triggering. 

## **Era 4 — InternalError Hook** 

Hook the InternalError.name getter to perform cleanup/diagnostics when interrupted: 

```js
Object.defineProperty(InternalError, "name", {
  get: () => {
    console.log("Interrupted at", inx)
    inx -= 2    // roll back state
    return "HookedInternalError"
  }
})
```

_**Note:** the error still throws regardless. Interrupting inside the getter itself causes a stack overflow. Use only for state cleanup and diagnostics._ 

## **Era 5 — Generators** 

Generators are syntax-native state machines. `yield` creates safe pause points; `.next()` steps through cleanly. **Caveat:** any error inside the generator (including an interrupt) kills it permanently — so the goal is to ensure it _never_ interrupts while active. 

```js
let gen = function*(){
  let inx = 3
  while (inx < 1000) {
    let a = true
    for (let i = 3; i < inx; i += 2)
      if (inx%i == 0) { a = false }
    if (a) { console.log(inx) }
    inx += 2
    yield              // safe pause — 5 000 fresh IU next call
  }
}()
function tick(){
  while (1) { gen.next() }
}
```

## **Era 6 — eval() as Voluntary Interrupt Point** 

Tom added eval() restrictions to limit abuse. Unintended effect: eval() resets IU and may interrupt right there — making it a _controllable_ checkpoint: 

```js
safeCode()
eval()        // interrupt here OR reset IU
riskyCode()   // guaranteed < 5 000 IU → always completes
```

## **Era 7 — api.isNearInterrupt()** 

Separates the two concerns of eval(): _querying_ proximity to an interrupt vs _acting_ on it. Allows conditional voluntary pausing without the gamble of eval(). 

_Coming soon:_ _**api.isNearRatelimit()** — analogous self-throttling for the rate limiter._ 

## **7. Rate, Text & Item Limits** 

## **7.1 Rate Limit** 

Sliding-window model: N ms of work per Y-second block. If the allowance is exhausted mid-block, code cannot run until the block resets. Violation message: `Error: you must wait Z seconds before running code.` 

## **7.2 Text Limit** 

All text limits are evaluated via JSON.stringify: 

```js
function tooLarge(obj, limit){
  return JSON.stringify(obj).length > limit
}
```

Consequently the API only accepts and returns JSON-serialisable types — never class instances or internal engine objects. This is by design: sanitising through JSON.stringify/JSON.parse prevents users from subverting the API. 

## **7.3 Item Limit** 

Prevents world/server crashes. Enforced per-world (mesh, particle, mob counts) and per-API-call. No API is permitted to crash the world — such a capability could be exploited to lock owners out of their own worlds. 

## **8. Outro** 

This essay represents the collective knowledge of the Bloxd coding community as of 27 May 2026, made as factually correct and comprehensive as possible. Errors and amendments are welcome. 

**Happy Bloxd Coding to you all — GlitchHunterCoder** 

