# ESSAY v1

# Bloxd Engine Essay of 30+ messages
Now that i got your attention, welcome to my biggest ever essay ive ever made on bloxd, it is soo huge that it took 1 and a half days to write, and takes up 30+ messages in dc just to display it all
# Intro

Now i may somewhat state stuff which is already known, or is obvious, that is because im making sure that this essay is as compehensive as possible, this essay hopes to contain all info about bloxd engine and how code is executed and how code is ran, and so below here is the essay

# Globals
## Static
Bloxd has multiple globals in bloxd io api, the following listed below
```js
[
  "Object","Function","Array","Number","Boolean","String","Symbol","RegExp", // Natives
  "Error","EvalError","RangeError","ReferenceError","SyntaxError","TypeError","URIError","InternalError","AggregateError", // Error Types
  "parseInt","parseFloat","isNaN","isFinite", // simple functions
  "decodeURI","decodeURIComponent","encodeURI","encodeURIComponent", // URI functions
  "escape","unescape", // escape/unescape
  "Infinity","NaN","undefined", // primative values
  "Math","Reflect","eval","globalThis","JSON","Proxy", // Misc
  "Map","Set","WeakMap","WeakSet", // Maps/Sets
  "ArrayBuffer","SharedArrayBuffer","DataView", //Array Buffer functions
  "Uint8ClampedArray","Int8Array","Uint8Array","Int16Array","Uint16Array","Int32Array","Uint32Array","BigInt64Array","BigUint64Array","Float32Array","Float64Array", // Typed Array
  "api","console","Date","myId","playerId","thisPos", // Injected lot
  "Symbol(Symbol.toStringTag)" // Symbol, oddly enough if you use Reflect.ownKeys(globalThis) //all symbols show up as null, i doubt that is intentional
]
```
as well as some accessible instrinics
```js
GeneratorFunction // function(){}.constructor
Generator // function(){}().constructor
```
as well as small amount of support for `async`
```js
let AsyncFn = async function(){
  console.log("called") //not ever possible to get here
}

// AsyncFn.__proto__ //undefined
// AsyncFn.prototype //undefined
// AsyncFn() //Error: Not a Function
// AsyncFn //function
// AsyncFn.name //"AsyncFn"
// AsyncFn.length //0
```
[Fig 1]()
but all of it is shown in Fig 1 (note that Fig 1 isnt completely accurate to its naming or values stored inside but it shows general structure)
some interesting notes from Fig 1 is the following
```js
- Most of the methods which exist here, are very old methods, which were made deprecated along time ago, methods like the following
  - String.prototype.anchor 
  - String.prototype.big //this list for string goes on for a long while, skipping
  - RegExp.prototype.compile
- some props exist which are not accessible or even usable, but still exist none the less
  - Function.prototype.fileName
  - Function.prototype.lineNumber
  - Symbol.asyncIterator
- some which exists but has no explaination of how to even use it in specifications anywhere
  - Symbol.operatorSet //couldnt even figure out what this does
  - Object.__getClass //looks to get some classes when called but thats about it
- api specifically has no length given
  - api["getBlock" /*Any_Api_Function*/].length //always 0
```
so thats what globalThis looks like, but thats not the only globals, previously i described the `Injected` Lot, now ill show why it has been given that name
for this lot instead of being just set at startup, appears to be looked at every time code executes, it does more GETS then that but that will be demonstrated in a future chapter
## Injected
Fig_2.png
```js
SET: api = [object Object]
SET: console = [object Object]
SET: Date = [object Object]
SET: myId = -1
SET: playerId = -1
SET: thisPos = 0,0,0
GET: api
GET: console
GET: Date
```
this image here has been produced by proxying globalThis and watching all that tries to access it, we note what is shown above is how it is all set up, we can take a look at what it does
```js
| Operation | Keys                                | Presence                         | Description                |
|-----------|-------------------------------------|----------------------------------|----------------------------|
| `SET` ?   | [`...allCallbacks`              ]   | all or nothing                   | sets engine-side callbacks |
| `SET`     | [`api` , `console` , `Date`     ]   | always                           | sets necessary values      |
| `SET`     | [`myId`, `playerId`, `thisPos`  ] ? | each independently optional      | sets code block specifics  |
| `GET`     | [`api` , `console` , `Date`     ]   | always — `Date` signals boot end | checks necessary values    |
```
so from this we can see many points
```js
- It accesses all callbacks (that would be the magic behind world codes working, after first run of world code, it GETS those callback names and puts them inside the loop
  - from there it remains closures with world code
- It ALWAYS SETS api, console, date: not too sure why always that but does it to update the api, console, and date (not too sure why every time tho?)
- It Sometimes SETS any number of myId, playerId, thisPos: for code block specifics, if it is needed or not
- it then ALWAYS checks what was always set api, console, date: my bet is its the engine checking if everything loaded correctly and if it didnt it returns a 
  - `Error: Please Report to Developers on Discord` (consider this my report lmao)
  - but its rather annoying when the engine then decides because some invisible invariants were broken, it just does that throw
  - i would HIGHLY love it if it would instead just let me, after all if the environment breaks, i am doing that willingly, not unintentionally
```
Fig_3.png
<img width="1097" height="1079" alt="image" src="https://github.com/user-attachments/assets/7113f841-5c5c-4bdf-8cf1-25ddca64d72e" />
This is the graph ive built which shows the order of execution and how things will execute in bloxd as a whole, now this graph may look much, and it is, ive made it as simple as i can but ill go through each one in the chapters which follows
# Flow of Execution
## General
first a rundown of common errors which we get, which ill get to which explains how it works, i forced errors to return their stacks, and here is what it gives

- Syntax Errors
```js
/
```
will return
```diff
- SyntaxError: unexpected line terminator in regexp
-     at eval.js:1
//note the eval.js:1, that 1 is line numbers
```
- Constructed Error
```js
throw new Error("Custom Name")
```
```diff
- SyntaxError: unexpected line terminator in regexp
-     at <eval>
```
notice the difference
- Funny QuickJS died Errors (ik this isnt related but o well these are funny)
```js
let c = 1;
let count = 3000;
let ex = 'if(c)c;'.repeat(count);    
eval(ex);
```
used to give
```js
Aborted(Assertion failed:label_slot[i].first_reloc==NULL, at: ../../vendor/quickjs.c32944,resolve_labels) //report 2 now lmao
```
before the eval patch you made which resets IU
now back on topic,
- Numeric Lines
```js
new Function(`let a={};a.b.c`)()
```
becomes
```diff
- TypeError: cannot read property 'c' of undefined
-     at anonymous (<input>:3)
//note the 3
-     at <eval>
```
now that is odd since it was on line 1, so where ever code is being executed its at line 3, so evaljs must be structured
```js
//not here
//not here
eval(`{${code}}`) 
```
now from there we experimented and found the following
```js
  let varLocal = "i live in this scope"

}; //NOTE HOW THIS IS A CLOSING BRACKET FIRST

let varGlobal = "im still standing..."

{ //THEN AN OPEN BRACKET

  console.log(varGlobal) //works
  console.log(varLocal) //fails
```
but even curiouser is the following, if i run that code in 1 code block
```js
//BUT THEN USE AN ENTIRELY DIFFERENT CODE BLOCK
console.log(varGlobal+"!!!") //"im still standing...!!!" //its here
```
and that is frankly AMAZING, a way to declare a globalVar without globalThis, how come this isnt standard, its would be SOO useful, anyways, back to essay

that tells us exactly how code seems to run, its similar to this fashion for 1 code block
```js
{
  //#BEGIN CODE HERE
  console.log("my code neatly inside here")
  //#END CODE HERE
}
```
so when we pull that escape trick, it is equivilant to as follows

```js
{
  //#BEGIN CODE HERE
  let varLocal = "i live in this scope"

}; //NOTE HOW THIS IS A CLOSING BRACKET FIRST

let varGlobal = "im still standing..."

{ //THEN AN OPEN BRACKET

  console.log(varGlobal) //works
  console.log(varLocal) //fails
  //#END CODE HERE
}
```
which now clearly shows why this worked, we forced it to escape outside the scope, thus declaring local global vars which is REALLY useful (since i dont like having to share values constantly through globalThis for anything to be transferred) and we can go even crazier, what about `var` or hoisting, where does that come into play, well now we get to see it
```js
//this might be common knowledge but im still stating it here for my sake

- DECLARE: let/const
  - BLOCKER: blocks
  - HOIST: name only -> TDZ (unusable until declaration line)

- DECLARE: var
  - BLOCKER: function boundaries
  - HOIST: name only -> undefined until assignment

- DECLARE: function
  - BLOCKER: function boundaries
  - HOIST: name + value (hoisted to top of block)
```
And so the rules then play out as follows
```js

/** Before Global (BG)
 * scopeLet Error
 * scopeVar - undefined
 * scopeFnc - undefined
 */ 


;{ //SCOPE
  /** Before Local (BL)
   * scopeLet Error
   * scopeVar - undefined
   * scopeFnc - function
   */ 

    let scopeLet = 1
    var scopeVar = true
    function scopeFnc(){}

  /** After Local (AL)
   * scopeLet 1
   * scopeVar - true
   * scopeFnc - function
   */ 
};


/** After Global (AL)
 * scopeLet Error
 * scopeVar - true
 * scopeFnc - function
 */ 
```
So this shows a way of usage rules, now we can compare and ask the difference between how global scope is altered when running code
- what if we are forced to only have inside SCOPE (BG & AG) [since vars have to break out of their code block scope]
- what happens if we are allowed to go outside scope (eg if SCOPE IS global scope) (BL & AL) [in that scenario SCOPE IS global scope and the vars are already broken out of code blocks]
and see any differences between our tests, and what we find is below
```js
- Before Differences
  - scopeFnc
    - Global: undefined
    - Local: function //not too much practical use, since that would require travelling backwards to a code block which has already ran and executed
- After
  - scopeLet
    - Global: undefined
    - Local: 1 //this IS useful because it allows us to define globally scoped variables without globalThis, this is a feature not usually achievable without this escape method, but thanks to it, is very useful indeed
```
This shows the hidden feature of global local variables which without this escape hatch methodology, would be completely impossible, but with it is a key part of variables
## Execution steps
Now that we have shown how is wrapped, now to demo how it executes, now im not going to write up a full JS spec this engine conforms to that would take me a long time to do, but ill run down the basics here then explain more about the limiters and other factors in future chapters

Now Fig 3 is going to come in useful as that is a map of our understanding of the execution model as far as im aware of it in this chapter imma go through the function below and its process

> `Step_Code(){ ... }` 
- Does a rate limit check to start off with
  - i won't explain what a rate limit check does, that is reserved for the Limiters chapter, same with all the limiters
- if that passes okay, then it will reset the IU count back to 0 and begin the `ByteCode` process
- then it checks if code is done, if so, it ends, else it does an Interruption Check and only if that passes does it go back round to do another ByteCode instr

> `ByteCode`
- will run exactly 1 ByteCode instructions from the compiled QuickJS code
- will then update IU (Interruption Unit) and RT (Runtime) as follows
  - IU: if it is in a ByteCode instr whitelist it will not increment IU at all (hence some ops being 0 IU) (i say whitelist due to time when IU temporarily died due to early release of eval new behaviour, where either the whitelist was forgotten, or the code was counting outer eval of our code) otherwise if it is not in whitelist it increments by 1
  - RT: either added up after the instr or (the more likely option) is its calculated every Interruption Check or Rate limit
now that is the general purpose of what code does to execute, however there is more to this, specifically the Initialisation process and the Return value process, those are laid now next
## Code Blocks
Now code blocks do that same process as laid out before but they do extra
Firstly they handle initialisation of the environment, that being variable injection to setup code block vars and setup verification (shown previously) before running the Step_Code() component, and afterwards after code is ran it takes the return value JSON.stringifies it and returns as output (JSON.stringify is a big theme of the engine, and ill dedicate a whole section just to that since it is a core part of the engine it will be noted down in limiters)
simple and efficient that process is but code blocks have another feature
### Chaining
Now we got close to figuring it out but thankfully Slushie showed us how it works, and it works as follows, (slushies statement transpiled into code)
```js
done=[] //already executed blocks
stack=[] //waiting to be executed
Chain(pos){
  try{
    let output = exec(pos) //if runs successfully, continue on, general name for execution of board or code block
    let [x,y,z] = pos
    let list = [
      [x+1,y  ,z  ],
      [x-1,y  ,z  ],
      [x  ,y+1,z  ],
      [x  ,y-1,z  ],
      [x  ,y  ,z+1],
      [x  ,y  ,z+1],
    ] //priority ordering
    list.forEach(item=>{
      if(
        !stack.includes(item) && //alr on list
        !done.includes(item) //alr done
      ){
        stack.push(item)
      }
    }) //makes sure only non done or waiting are added (eg they are new boards)
    sendReturn(output)
  }catch(err){
    sendError(err)
  } finally {
    let next = stack.pop() //take next one off the queue
    done.push(next) //since next one to be executed is now in done list
    if(next !== undefined){
      Chain(next) 
    }
  }
}
```
Now it looks complicated but all it encodes is that
- it recursively activates them one by one in a stack like fashion
now this allows numerous amounts of behaviour, like 
- chains (all have to succeed in order for an outcome to happen),
- branching (a block will behave differently based on if a previous one succeeded or failed)
- branch slicing (in a branch use press to execute to execute a code block later in branch thus snipping off that branch to further executes)
- and alot more
now usually this is easily handled or made redundant by code blocks specifically try catches but they have their uses,
- within survival mode using press to buy/sell from chest (btw is there a difference between buy sell other than just names),
- pressing of code blocks to load code either just as a chain 
- dynamically handling errors, 
- using it to execute code automatically after fatal error like interruptions (yes interruption)
  - by using branching (eg you just place 5 code blocks in plus formation, in the outer 4 put `while(1){}` and in the center, place just `1` inside, then press center)
  - it can execute that code then for all the branches it would interrupt but due to branching it continues anyways, allowing for a way to use code blocks to handle interruptions
- and again alot more
So thats what code blocks have got, now for the chart

## World Code
World code follows the same logic as code blocks to a certain extent, but it doesn't have 
- chaining
  - only 1 world code exists so it cant chain, doesnt even have a block to which to chain with
- displayed return value 
  - that is replaced with the callback registration process which either 
    - returns `Found ${number} callbacks: ${array.join(", ")}`
    - throws `Found no callbacks`
  - oddly it still enforces return too large limit still (more details later)
Now code execution for the first run is identical to code blocks, that part doesn't change what changes is what it does next specifically
> `WorldCode`
- does first run of code using `Step_Code()`
- gets all callback names avaliable (Fig 4 below)
- LOOP:
  - the callback stack is filled up during gameplay
  - call each callback once, per time an event occured (eg onPlayerClick, will run as many times as clicks happened with specific args)
  - after all callbacks have been called which were needed, wait 50ms before running again

and thats how world code callbacks work, they repeat this process until world code is refreshed or lobby close
Fig_4.png
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
now for some specific details about how it works, first

### Callback Extraction
After it runs world code for first time, it then does a GET on all callback names and checks if the result is either
- undefined //isnt defined (tester by item === undefined)
Or
- function //is defined (tested by doing item() and seeing if it works)
If that is the case, it is allowed, if not then it logs back
- `${CallbackName} is assigned to a non-function`
Not an error (it is nice that it doesn't throw an error)
Afterwards it will take those callbacks and place them in the callback loop

Callbacks in this sense can be thought of as an event loop/listener, each time a specific event occurs, the event listener will be triggered

now when they are copied over, you cant then edit the callback mid execution usually, as shown by code below
//FORMAT, WORLD CODE, CODE BLOCK
```js
tick=()=>{console.log("old tick")} //copies this
```
```js
tick=()=>{console.log("new tick")} //doesnt edit it, was already gotten and copied along time ago
```
here it was copied, so it is unable to be edited later
```js
Ref=()=>{console.log("old tick")}

tick=Ref //copied again before done
```
```js
Ref=()=>{console.log("new tick")}
```
still copied before editable
```js
Ref=()=>{console.log("old tick")}

tick=()=>Ref() //is computed each time
```
```js
Ref=()=>{console.log("new tick")}
```
worked, serves as a temporary callback wrapper while it waits for real code

so thats why code loaders are in high demand always, not because of just world code space, but because it serves as a temporary callback meaning you can decide whenever you like what callback should be instead of being forced to decide within the first tick otherwise it isnt captured, and within almost infinite space of code blocks from which to organise your code into, it is just better suited for the purpose than using callbacks alone, and thanks to that they have been made
- BREAKING NEW DISCOVERY: 
this was found as the essay was being made, specifically a definition of how callbacks are extracted precisely
```js
let someOtherFn = () => {console.log("old")}


onPlayerJump = 123 //not a function

Object.defineProperty(globalThis, "onPlayerJump", {
  configurable: true,
  get() {
    return someOtherFn; //is ONLY thing callbacks care about
    //copies object reference -> () => {console.log("old")}
  }
});

void 0 
```
when extracting callbacks, it is found that all it does is that it takes a reference copy of the returned GET (Reference as in Objects are References, not variable references (noting that since even i got confused))
- to demo that idea
```js
someOtherFn = () => {console.log("new")}
//wont edit the callback, it is still holding old object reference
```
so that is specifically how callbacks are hooked onto code
### Event Loop
here each callback which was registered now is looping
- LOOP: 
  - Each callback event is gathered
  - Each callback is triggered
  - after all is done, it then waits again 50ms

so with callbacks, and code execution as a whole, a very important rule which is known
- Date.now is cached once per tick
- each callback has a fixed order of execution
using that you can make the code below
```js
let isNew = () => { //checks if now in certain callback it is a new cycle
    if(LAST !== Date.now()){ //if cache changed, new cycle
        console.log("=== NEW LOOP ===")
        LAST = Date.now() //set to new
    }
}

onBlockStand = () => {
    isNew()
    console.log("stand",Date.now())
}
tick = () => {
    isNew()
    console.log("tick",Date.now())
}
```
this is the basic methodology to find the ordering of callbacks, as for how each callback is called, its no different to ordinary function call `fn()`
running this code shows that onBlockStand runs BEFORE tick, and so similar logic can be used for other callback to detect their ordering, 
generally this approach can be used
```js
let LAST = Date.now() //cache
let list = []

let isNew = () => { //checks if now in certain callback it is a new cycle
    if(LAST !== Date.now()){ //if cache changed, new cycle
        console.log(list)
        LAST = Date.now() //set to new
        list = [] //reset
    }
}

let CALLBACKS = ["onBlockStand","tick","onPlayerClick","onPlayerClickUp"] //add more here

CALLBACKS.map(item=>{
    globalThis[item]=()=>{
        isNew()
        list.push(item) //add to list
    }
})
```
now if we look at Fig 5 below that shows us something, these callbacks contradicted themselves, sometimes it says that it does click first, other times click up, but its just those
well thats because that develops a new theory, callbacks are executed in a fixed order, yes, but perhaps async is used for the engine to decide how to schedule it (so although push happens, it could be delayed)

Fig_5.png
```js
Log: ["onBlockStand","tick","onPlayerClick","onPlayerClickUp"]
Log: ["onBlockStand","tick","onPlayerClickUp","onPlayerClick"]
```
now it has been confirmed by Tom that 
- the order is kinda random
-  Date.now is updated at end
- and code block execution within this ordered list is random
so that previous theory could very well be how it works, and it would be useful as well
now onto next
# Limitations
bloxd has a whole HOST of limitations but it generally comes down to the following
- Interruption Limit
  - (5 000 IU)
  - (Runtime)
- Rate Limit
  - Code (N ms of code per X seconds ; up to 7 seconds)
  - Board (same as Code)
  - eval (same as Code)
  - Chunk loading (up to 1 minute)
  - Block Data setting
  - Code collab
- Text Limit
  - Character (WC)
  - Character (CB)
  - Chunk (schematic)
  - Chunk (exported world)
  - 2k character (function arguments)
  - 2k character (item attributes)
  - 10k character (lobby db)
  - 10k character (player db)
- Item Limit
  - 200 mesh (without physics)
  - 50 mesh (with physics)
  - particles
  - mob
## Interruptions Limit
now this would be just about explaining interruptions, but i also got to satify a favour, so HISTORY LESSON on anti interruption but first explaining what they are
### What are they
this feature is built into QuickJS itself, since bloxd turns our code into ByteCode it can set specific interruptions conditions in bloxd case the condition is as follows
```js
if (IU % 5000 == 0 && TU > MAX_RUNTIME_ALLOWED) {
  interrupt()
}
```
which means it relys on IU and TU (Interruption Units (specific unit which is measurable and which determines when bloxd will interrupt) and Temporal Units (just a unit of time, like seconds or minutes, but defined in terms of code which doesnt rely on Date.now (due to it being cached) in order to time) )
many an essay has been written on such a subject, and imma add to it, BUT imma combine all essential topics from the different eras of anti interrupt and how we developed new technology for anti interrupt
### Terminology
- Idempotent (running a function multiple times had the same effect as running it once)
- Atomic (can EITHER fail with no side effects OR complete fully)
- Weak Interruption Safety (when interrupted, breaks execution of current code, but it guarantees that future ticks or operation dont break as well (eg it isolates each interruptions effect to only the current operation) [is Atomic, but not idempotent])
- Strong Interruption Safety (when interrupted, can recover execution of CURRENT code, as well as guaranteeing that future ticks dont break as well (eg interruptions are isolated per tick, and ticks can repair themselves as to not break) [Atomic AND idempotent])
### History on Anti Interrupt
before we do it, here is a piece of code here so then you can see it travel through time, 
```js
let inx = 3
while(inx<1000){
  let a = true

  for (let i=3; i<inx; i+=2) {
    if(inx%i == 0){a = false}
  }
  
  if(a){
    console.log(inx)
  }
  
  inx+=2
}
```
- Anywhere Interrupt (basically everywhere could be interrupted so you need to guarantee that it has ran all needed code before moving to next step)
this was the starting point of anti interrupt research, back then we didnt know ANYTHING about how it works, so the general consensus was that it could interrupt anywhere, so you had to make sure that your code, no matter what could handle an interruption anywhere, and be able to continue without issue, it was a tall order, but it was done
```js
let inx=null
let i=null
let a=null
start=0
cmd=null
function tick(){
  if(cmd==null){
    start=0
  }
  if(cmd=="prime"){
    if(start==0){
      [inx, i, a]=[3, 3, true];
      start=1;
    }
    if(inx>=1000){
      cmd=null
      return
    }
    if(start==1){
      if(i>=inx){
        start=2;
        return
      };if(inx%i == 0){
        a = false
      };i+=2
      return
    }
    if(a){
      console.log(inx);
    };
    [a,i,inx]=[true,3,inx+2];
    start=1
  }
}
```
```js
if(start==0){cmd="prime"}
```
The key features which made this idea possible was that you never moved on from a piece of code until it was ALL done, and that was the guarantee, built on 
`Idempotence and Atomicity`
although those terms would not be brought up for a while longer until sulfrox essay, now this did use some state ideas this would develop into the next category, but this is where it all started, and fun fact, back when eval broke and removed IU and interruption safety, this methodology would still function correctly since it operated under the worst case scenario of interruptions (that it could happen anywhere) as you will soon learn, that was not the case for interruptions
- State machines (we keep track of state, and move between them ensuring that each states function can run by itself, then it does one transfer of state per tick)
This was the next natural development of anti interrupt, since it was developed that if you had several pieces of code that was themselves strongly interruption safe, and had a clear map of how to move between them, guaranteed that the whole program was strongly interruption safe, this did gain some traction and then remains a popular method of interruption safety to this day, perhaps because it covers a very broad general idea
```js
let state = 1
let inx=null
let i=null
let a=null
let Atoms = [
()=>{ //Null fn
},()=>{ //initialisation
  [inx, i, a]=[3, 3, true];
  state=2;
},()=>{ //inner loop
  if(inx>=1000){state=0;return};
  if(i>=inx){state=3;return};
  if(inx%i == 0){a=false};
  i+=2;return
},()=>{ //outer loop
  if(a){console.log(inx)};
  [a,i,inx]=[true,3,inx+2]
  state=2
}]
function tick(){
  Atoms[state]()
}
```
Here was the main idea, states, and since each code snippet is Strongly interruption safe, and there is a path between them, which is also safe, the whole code execution is safe, now this style lives on to this day, because of its versatility and its general ability to be used however you like, and the strength it guarantees was unmatched, now this would not change for a while, not until a curious behaviour was spotted, when Interruption stopped being avoided, ane began being measured
- Cost Reduction (we can measure IU (interruption units) and now we aim to reduce that as much as possible)
Here was when sulfrox posted his Interruption tester, and that was the very start of something bigger

code block A: ```js
//Note this code is for historical purposes only, there are since better tests shown below
globalThis.array = [1];
for(let i=0;i<12;i++)array=array.concat(array);
while(true){}```
code block B: ```js
let pre_computation_count = 1000;
globalThis.interruption_counter = pre_computation_count*3+1;

//insert any operation you need to time
JSON.stringify({1:1})
//between these 2 comments

while(pre_computation_count--)array.copyWithin(0,1);

while(true) {
    interruption_counter++;
}```
code block C: ```js
interruption_counter```
Now this tester showed that it was possible to test for interruptions, but we didn't really get that far with it, having to manually test each code was nothing but painstaking to do, so we didn't manage to go far with it, not until i took a simple idea and asked, "what if we tested in bulk using tick"
- Batch Testing (mass analyse stats of functions and calculate each and every cost, reduce cost by any means needed)
So i made that batch tester, and from that anti interrupt logic was in full swing, the 3 day long discussion day and night for 3 days straight between me, delfineonx, chmod, the_ccccc, dulph, and sulfrox on round the clock discussions and testing sending over 1000 messages back and forth, it was the MOST popular it has ever been, and after that discussion the subject lingered within discussions for up to a month were many coders optimised their code for IU efficiency which involves the following:
> - bitwise logic + element access over if statement
> - added together strings over template literals
> - assignment, deletion and changes to a variable all cost 0.
> - while loops over for loops?
> - IU addition happen when code needs to branch (ifs, while computed, for, ect)
> - new expression costs 1 IU
> - function calls cost 1 IU
From there cost analysis of functions were made alot more common, but there was one debate which lasted a long time
 - IU vs TU
```js
if(interruption_count % 5000 == 0 && runtime > MAX_RUNTIME_ALLOWED){
  interrupt()
}
```
Due to this golden rule above a big debate was sparked over whether optimising for fast runtime (TU) or low interruption points (IU) was the best way, and this debate never really got an answer to this debate because it tied down to do you want to protect against interruptions, or make sure it doesn't trigger in the first place
```js
let inx = 3
while(inx<1000){
  let a = true
  let i = 3
  while(i<inx) {
    a = !!(+a & +(inx%i == 0))
    i+=2;
  }
  [
    ()=>{},
    console.log,
  ][+a](inx)
  inx+=2
}
```
this uses some of the tools of the trade to get this done efficiently using minimal IU, note how most if statements are gone, and for loops been converted to while
- InternalError hook (using define property to hook onto the error itself to allow us to tidy state)
After that lot a new method arrived which almost felt like cheating, and to some extent it was, that would be the internal error hooking 
```js
let inx = 3
while(inx<1000){
  let a = true

  for (let i=3; i<inx; i+=2) {
    if(inx%i == 0){a = false}
  }
  
  if(a){
    console.log(inx)
  }
  
  inx+=2
}

Object.defineProperty(InternalError,"name",{
  get:()=>{
    console.log(
      "Interrupted",inx,a,i
    )
    inx-=2 //reset if needed
    return "HookedInternalError"
  }
})
```
now this code i cant suitably turn into this eras format but for more complex codes it is perhaps useful for resetting stuff or knowing that it got interrupted and pulling back
> - DISCLAIMER
This does NOT mean you can DDoS bloxd itself, there are limitations after internal error, the error still throws no matter what, and if interrupted inside itself either gives a stack overflow or breaks since you didn't specify a return value for the getter

Now this methodology has genuine usage, namely delfineonx interruption framework, where the core of it is based on state machines and this hook method
Now from there we got discoveries alot quicker which can be considered late stage anti interrupt
- Generators (we can use yield; to pause the code, serving as state machines but alot better, downside is that if it ever interrupts, you cant recover)
Generators were considered before but not too useful for reason later down the line, now from here the method looks the same throughout just with some stuff commented or un commented, this code is for all remaining chapters of history of anti interrupt
```js
let gen = function*(){
  let inx = 3
  while(inx<1000){
    let a = true
  
    for (let i=3; i<inx; i+=2) {
      if(inx%i == 0){a = false}
    }
    
    if(a){
      console.log(inx)
    }
    
    inx+=2
    yield; //pause point
  } //Long task
}() //Code container

function tick(){
  //if(api.isNearRatelimit()){return} //if near ratelimiting, look again next tick

  while(1){
    //if(api.isNearInterrupt()){return} //if near interrupting, do next tick instead

    //eval() //if not near, reset IU, and go again

    gen.next() //executes next part, //now with 5k fresh IU, and confident TU

    //return //if you dont want it to run more than once per tick
  }
}
```
Now generators were extremely powerful for this job because it naturally syntactically converts your code to a state machine with a built in `.next()` to allow stepping through your code easily and simply, combine that with yield ability to take IN input and give temporary output, its potential was unbounded, but as all powerful tools do, it has one Achilles heel, that being if it ever errors (or in this case interrupts) while inside, the entire function dies, so the implementation of the yields power was built for 1 purpose to make sure that it NEVER interrupts while inside, and thats what these final chapters and additions give
- eval() (Tom added a feature to try prevent us from spam calling it (eval() will reset IU count, and may interrupt right there and then)
This addition came because tom wanted to limit how much people called eval over and over since it was effecting servers, and so after a bit of a mishap where the environment completely ditched IU, sending us back to the stone age (Idempotence age) before being fixed, and so after that was added an unintended side effect of that is that ironically, people decided to use it MORE often, due to it having one neat ability, when `eval` is called it will reset IU points and maybe interrupt right there and then, this naturally got turned into a voluntary interruption point, where coders could use it to reset IU points for a risky operation or incase you wanted to stop there but didn't know if it was safe, one such pattern was this
> Risky Code
```js
safeCode()
eval() //interrupts now or resets IU
riskyCode() //if always under 5k IU, guaranteed to run and complete
```
Which was revolutionary for interruption safety, and which shows why it gained so much traction in code usage, because of that simple idea, allowing us to not have to make guesses or manage IU in a safe manner, but instead control IU directly, manipulating the very unit used to control interruptions, and phase lock it into a specific code to control where the risk is and guarantee that parts of code within tick will never interrupt ever
- api.isNearInterrupt() (new api allows us to track when interruptions are likely and abort safely, a better version of eval)
this served as a safer alternative to eval, instead of having to ask interruptions were likely and reset IU, and potentially get interrupted all at once, those purposes could be separated, now you could ask if a particular code was near interrupted with one function, and voluntarily interrupt/reset IU with another, so this was a very welcome addition
- SOON: api.isNearRatelimit() (with this should allow self throttling code)
this is the final api im waiting on before we can make self throttling code since we can apply the similar logic to rate limiter (yes ik this is about interruptions, but this is the only history segment im putting in) in order to throttle code under both rate limiter AND interruptions, now to wrap this lot up with the last lot of limitations
## Rate Limit
Rate limits are used all over bloxd as a whole but generally it is to stop too many requests, now i don't have too much about rate limiters here tbh, since the way they work is rather simple, it is based on a `X seconds of work every Y second block` so an example, lets say we allow 1 second of work every 10 second block, then if a code
- runs from 0-0.5s //0.5s
- runs from 1.5-2s //0.5s
then all work allowed is done, and from 2-10s, no more code can be ran, and any attempt to do so will result in a `Error: you must wait Z seconds before running code` where Z is number of seconds until the next block, this process is similar or identical through all rate limits present in bloxd
## Text Limit
For these limits, they often use JSON.stringify to decide, eg args too large, return too large, schematic too large,  ect ect, it is always JSON.stringify behind it, the method is quite simple 
```js
function tooLarge(obj,limit){
  return JSON.stringify(obj).length > limit //thats all there is to it
}
```
Infact that is why all api follows a strict set of rules which is that api arguments will only accept both in arguments, and in returns, items which can be stringified (booleans, string, number, null, undefined, object, array) which may sound like i just named everything but critically the api will never return stuff like classes, or instances of a ChunkClass for example, that is to prevent us from breaking their api, by sanitising all user inputs and returns through JSON.stringify or JSON.parse
# Item Limit
Item limit is in place to prevent us from crashing bloxd itself, this is done through preventing us to do certain things with api, either through world limitations: too many particles mobs ect, to make sure we dont crash their servers, or through api arg limitations, cant set value too high, api.openShop is rate limited (yes it somewhat falls under here since it is limited separately) or other means, since it is probably under api rules that no api is allowed to crash world, because if so, it can be maliciously used to lock someone out their own world, and so that would be not good for who this is used against so it is preemptively a rule that no api can crash
# Outro
And so that is the end of the essay, i hope you like all the info inside, i have tried to make it as factually correct, and comprehensive as possible, if there are any errors i would be happy to amend it within the essay itself, and with that i would like to sign off
- Happy Bloxd Coding to you all
- GlitchHunterCoder
# FORUM MAIN POST OUTRO

with that, that is the entire essay sent and posted, that took me like 1 and a half days to write, 

# PING LIST
- Pinged due to either being part of history mentioned, or because they woud appeciate it
  - `(Ocelote) Solve 3+3/3*3/3-3/3/3`
  - `Tridentify`
  - `delfineon`
  - `the_ccccc`
  - `DreamWaker (Realm Defenders)`
  - `LT12PVPER#WSLUSHIE`
  - `Sulfrox`
- Pinged because they requested
  - `!ABCDEFGHIJKLMNOPQURSTVWXYZ` x 10 -> https://discord.com/channels/804347688946237472/1341451374868693053/1509600349076455619
  - `Crownix (join bloxdhub.com)` -> https://discord.com/channels/804347688946237472/1341451374868693053/1509600411907260467
- Pinged due to being annoying (joke entry)
  - `Bloxd Fan` -> blocking my pings

# Honorable mentions
- Tom -> he allowed me to make the essay in his dms, and thus inspired me to make this whole thing











