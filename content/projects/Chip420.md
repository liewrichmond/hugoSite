+++
author = "Richmond Liew"
title = "Chip420"
date = "2020-10-30"
description = "An automation project"
tags = [
    "frontend", "emulation", "Typescript", "React"
]
+++

tl;dr: Computer architecture is dope af. [Source](https://github.com/liewrichmond/Chip420) | [Demo](https://obscure-meadow-66386.herokuapp.com)
<!--more-->

### The Final Product
![sample](/images/chip420.gif)

### But Why tho

Digital Logic and Design and Computer Architecture were two of my favourite classes while I was at Northeastern(Shoutout to Prof.Tiwari and Prof.Ubal!). What really stuck out to me about those classes is:

1. How complex programming semantics are broken down into tiny instructions
2. The CPU doesn't really support all that many instructions at all

With the fundamentals laid out to me, I decided to try my hand at emulation and really putting my knowledge into practice. I did some digging and found that CHIP-8 was a really good starting to get in to emulation for beginners. I decided to use Typescript + React for this since I really wanted an easy way to distribute and showcase my project and I also thought the contrast of using a language so far abstracted from metal to implement a bare metal component was interesting.

### The Code
As always, nothing makes me happier than a nice clean project structure. So in order to keep things neat, I decided to hide the main logic in a `resources/` folder and the rendering React stuff in a `components/` folder. `resources/` also kept a bunch of other assets like roms and such.
```bash
src/
|   >components/
|   |someComponent1.tsx
|   |someComponent2.tsx
|   >resources
|       >ts
|       |cpu.ts
|       |graphics.ts
|       >assets/roms
|       |tetris.ch8
|       |spaceInvaders.ch8
```

It's also worth mentioning that I kickstarted this project with `create-react-app`. It's seriously an amazing tool helped me get everything setup quickly and smoothly. Goodbye messing around with config files!

I'd also like to preface this by saying that the final code is here has been tweaked many times. I definitely didn't come up with all this in my head from the get-go. I kinda just reorganized things on the fly based on how the project changes.

#### Do yOu hAvE gAMes oN yoUR pHoNE?
With the main scaffolding setup, we can finally get to writing some code! The first step that I decided to take with this project was to see if I could first load a rom. Since the emulator is essentially an interpreter for the instructions, loading up an instruction to see what kind of data we're actually dealing with would prove extremely helpful. If we were working on something locally like a Python or C++ application, we could easily do some sort of `open(file)` equivalent. However, since we're on the web here, we need to use a `fetch()` in order to get the data.

I still wanted to keep everything neat so I actually wrote file paths as an import.
```ts
    import Tetris from "../assets/roms/Tetris.ch8"
    import spaceInvaders from "../assets/roms/spaceInvaders.ch8"
    import testRom from "../assets/roms/test_opcode.ch8"
    import testRom2 from "../assets/roms/BC_test.ch8"
```
This meant that the variable `Tetris` now referred to the path to the location of the Tetris rom! As such, we're able to use this like so:
```ts
    export interface gameMetadata {
        romLocation: string,
        keyBindings: { [customKey: string]: string }
    }

    export const getGameMetadata = (): { [key: string]: gameMetadata } => {
        return ({
            "Tetris": {
                romLocation: Tetris,
                keyBindings: {
                    //For later...
                }
            },
            //Other games
```
With a little extra helper code:
```ts
    private async getRom(romStr: string): Promise<Uint8Array> {
        const response: Response = await fetch(romStr);
        if (response.body) {
            const reader: ReadableStreamReader<Uint8Array> = response.body.getReader();
            const data: ReadableStreamReadResult<Uint8Array> = await reader.read();
            if (data.value) {
                const rawRom: Uint8Array = data.value;
                return rawRom;
            }
            else {
                throw (new Error("Corrupted ROM data"))
            }
        }
        else {
            throw (new Error("Could Not Fetch ROM"))
        }
    }
```
We can now call it in a nice clean fashion!
```ts
    await getRom(game.romLocation)
``` 
With some `console.log` statements, you could actually see the raw rom data in the console!

Of course, `.ch8` isn't a valid file extension in the eyes of the Typescript compiler. So I actually had to create a little `env.d.ts` that declares `.ch8` files as an module
```ts
    /// <reference types="react-scripts" />
    declare module "*.ch8"
```

#### Thanks, Mr.von Neumann
Almost all modern day computers are based off of the von Neumann architecture. This computer architecture proposed that we have a main control unit and Arithmetic Logic Unit aka the CPU, talking to some memory that holds some temporary information. The CHIP8 is no different. Thus, our CHIP8 class looks like

```ts
    class Chip8 {
        private cpu: CPU
        private memory: Uint8Array;
```

The CHIP8 has about 4KB of memory (4096 bytes) but not all of it us accessible by programs. The first 512 bytes were initially reserved for the interpreter so anything after that would be program addressable memory. In our case, since, we don't need to store the interpreter in memory, not all of the 512 bytes would be used. The one thing that we do need to store in there, however, are some built in sprites. The CHIP8 had some default sprites/fonts stored in there and in case it was referenced somewhere, we wanted to make sure that it was there. I'm not going to go into too much detail about the sprites, but you can read about it [here](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM#font).

I wanted to ensure that the memory was in a known state upon initialization, so in the `Chip8` constructor
```ts
class Chip8 {
    private cpu: CPU
    private memory: Uint8Array;
    //other fields

    private loadFontsIntoMemory() {
        //Load default Hex fonts into memory
        const fonts: number[] = [
            0xF0, 0x90, 0x90, 0x90, 0xF0,
            0x20, 0x60, 0x20, 0x20, 0x70,
            0xF0, 0x10, 0xF0, 0x80, 0xF0,
            0xF0, 0x10, 0xF0, 0x10, 0xF0,
            0x90, 0x90, 0xF0, 0x10, 0x10,
            0xF0, 0x80, 0xF0, 0x10, 0xF0,
            0xF0, 0x80, 0xF0, 0x90, 0xF0,
            0xF0, 0x10, 0x20, 0x40, 0x40,
            0xF0, 0x90, 0xF0, 0x90, 0xF0,
            0xF0, 0x90, 0xF0, 0x10, 0xF0,
            0xF0, 0x90, 0xF0, 0x90, 0x90,
            0xE0, 0x90, 0xE0, 0x90, 0xE0,
            0xF0, 0x80, 0x80, 0x80, 0xF0,
            0xE0, 0x90, 0x90, 0x90, 0xE0,
            0xF0, 0x80, 0xF0, 0x80, 0xF0,
            0xF0, 0x80, 0xF0, 0x80, 0x80,
        ]
        for (let i = 0; i < fonts.length; i++) {
            this.memory[i] = fonts[i]
        }
    }
    
    constructor() {
        this.memory = new Uint8Array(4096)
        this.loadFontsIntoMemory();
        this.cpu = new CPU(this.memory);
        this.keyboard = this.createDefaultKeyboard()
    }
```

You'll notice that the `CPU` class also takes the memory as an argument for its constructor. This is done because we'd need to read and write to it during operations but also because I didn't really see loading a ROM as a "job" for the CPU. It does kinda skeeve me out that no one object owns the memory completely, but unless I wanted to write keyboard inputs and IO into the CPU class, I didn't really see a way to do it.

#### The CPU
The CPU has a couple of moving parts to it so I'mma break it down here

#### Registers
On top of memory, a processor also has some the ability to remember small bits of information temporarily by way of `registers`. Since a register lives within the CPU, it runs on the internal clock speed of it and allows for very quick reads and writes. The CHIP8 has 16 registers and an extra one called the `index register`. The index reg is typically used as temporary storage for arithmetic operations. For instance, if we'd want to calculate the address of a memory offset, the calculation would look like `memory address + offset`. But now if we need to access this bit of memory, we'd already forget it by the next cycle! Therefore, we'd store this into the index reg and load up that memory in the next cycle.

In terms of implementation we're just gonna use an array to represent the registers. It stores some data and gives us the ability to access them by index.
```ts
class CPU {
    private regs: Uint8Array;
    private indexReg: number;
    private pc: number;
    //other fields...

    constructor(memory: Uint8Array) {
        this.regs = new Uint8Array(16);
        this.indexReg = 0;
        /*Our adressable memory space starts at 0x200 cause
        the first 512 bytes are allegedly being used by the interpreter.*/
        this.pc = 0x200
        //other assignments...
```

It's kinda weird that we're able to take a specialized high speed bit of memory and just storing some value into OUR computer's memory, but hey that's the beauty of technolgical advancements!

Aside from the "storage" registers, there's also another specialized register called the `program counter` or `pc`. As the name implies, this register basically keeps track of where we are in the program (memory). After fetching an instruction from memory, we need to go to the next instruction, and thus we increment the pc! The question now is "By how much?". This depends on the "width" of our instructions. In the case of CHIP8, an instruction is 2bytes so therefore if we'd like to increment the pc after an instruction, we'd do `pc += 2`.

#### The Stack
Aside from the `pc`, our cpu also needs to keep track of where we are after we do a particular task. As an example, say we had a function `addOne(x:number){ return x+1 }`. when we run it, this function will essentially be placed into some region on memory and we'd need to adjust our `pc` to that address. Well, once we're done doing what we have to do in the function, we need to keep track of where we next! Thus, before we jump to the address of `addOne`, we'll put the next `pc` value onto the stack. 

But now, you must be wondering "Papa Richmond, how do we keep track of where we are on the stack?". With a stack pointer, of course! When we add something onto the stack, we increment the stack pointer, and when we're done, we just decrement it!

```ts
class CPU {
    private regs: Uint8Array;
    private indexReg: number;
    private pc: number;
    private stack: Uint16Array;
    private stackPtr: number;
    //other fields

    constructor(memory: Uint8Array) {
        this.regs = new Uint8Array(16);
        this.indexReg = 0;
        this.stack = new Uint16Array(16)
        this.stackPtr = -1
        this.pc = 0x200
        //other stuff
    }
```

#### Instructions
At its core, the CPU is really just a glorified calculator waiting for people to tell it what to do. As such, we need a contract to tell the rest of the world what the CPU is capable of, and what you can ask of it. This is done through an `Instruction Set Architecture` commonly referred to as an `ISA`. It just defines some methods that you're allowed to tell the CPU do. CHIP8 has 36 instructions in total. They're pretty straightforward to implement but can get VERY tedious. I won't talk about them here because there's plenty of information out there about the implementations like [Austin Morlan's blog](https://austinmorlan.com/posts/chip8_emulator/) and [THE go-to CHIP8 reference](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM#3.1).

#### Graphics and Rendering
Before we talk about the actual rendering, let's take a quick look at the draw instruction

```ts
// DRW Vx, Vy, nibble
private _Dxyn(opcode: number) {
    //Bitmasks to extract relevant information
    const spriteHeight: number = (0x000F & opcode)
    const Vx: number = (0x0F00 & opcode) >> 8
    const Vy: number = (0x00F0 & opcode) >> 4

    this.regs[0xF] = 0
    
    //Drawing Logic
    for (let row = 0; row < spriteHeight; row++) {
        const memoryAddr = this.indexReg + row;
        const spriteToDraw: number = this.memory[memoryAddr];

        for (let col = 0; col < 8; col++) {
            //account for line wrapping
            const yPos = (this.regs[Vy] + row) % 32
            const xPos = (this.regs[Vx] + col) % 64
            const shiftAmount: number = 7 - col
            const spritePixel = (spriteToDraw >> shiftAmount) & 0b1
            this.updateGraphicsAt(xPos, yPos, spritePixel)
        }
    }
}
```
At the start, we use some masks to extract the information. Then we start updating the pixels, row by row, column by column. Since the sprites are a default width of 8, we can just hard code it. You'll see that we call `updateGraphicsAt()` which looks like
```ts
private updateGraphicsAt(xPos: number, yPos: number, spritePixel: number) {
    const currentPixel = this.graphics[yPos][xPos];
    const newPixel = currentPixel ^ spritePixel;
    if (currentPixel === 1 && newPixel === 0) {
        this.regs[0xF] = 1
    }
    this.graphics[yPos][xPos] = newPixel
    if (this.renderer) {
        this.renderer.updateGraphicsAt(xPos, yPos, newPixel)
    }
}
```
Here, we grab the current pixel and then XOR it with the pixel we're trying to draw. If the pixel was initially 1 and it's now 0, we can mark a collision in our `0xF` register. The truth table for XOR is:

| A | B |XOR|
|:---:|:---:|:---:|
|0|0| 0 |
|0|1| 1 |
|1|0| 1 |
|1|1| 0 |

As you can see, when one bit is 1, the only way the output can be 0 is if the other bit is 1, denoting a collision.

Once, we've gone through all this, what we pass to the `renderer` will be exactly what we want to draw at that position so we can apply the changes

```ts
class Renderer {
    public updateGraphicsAt(xPos: number, yPos: number, pixel: number) {
        if (pixel && this.context) {
            this.context.fillStyle = "#00FF00"
        }
        else if (!pixel && this.context) {
            this.context.fillStyle = "black"
        }
        if (this.context) {
            this.context.fillRect(
                xPos * 10,
                yPos * 10,
                10,
                10
            )
        }
    }
}
```

Initially, I tried using Pixi.js to render the graphics but it didn't come out the right way. It's hard to describe exactly, but everything just looked misplaced. Pixels that should have been on were off and vice versa.

```ts
    public updateGraphicsAt(xPos: number, yPos: number, pixel: number) {
        if (pixel) {
            if (this.graphics.fill.color !== 0x00FF00) {
                this.graphics.beginFill(0x00FF00);
            }
        }
        else {
            if (this.graphics.fill.color !== 0x000000) {
                this.graphics.beginFill(0x000000);
            }
        }
        this.graphics.drawRect((xPos * 10), (yPos * 10), 10, 10);
        this.graphics.endFill()
    }
```

I couldn't figure out what happened, but I suspect that I might be trying to change the color a little bit too much...? Revisting the code, it actually might be worth to try adding another `PIXI.graphics` and have one dedicated to drawing on pixels and the other to draw off ones. It took a hot minute to figure out that it was just the actual rendering that wasn't working a not any of my rendering logic but I figured it out in the end.

#### Tying It All Together
Once I had the main components working, all I had to do was add some I/O and doing the actual game loop.

The key presses are pretty simple.
```ts
class CPU {
    this.keys = {
        0x0: false, 0x1: false, 0x2: false, 0x3: false,
        0x4: false, 0x5: false, 0x6: false, 0x7: false,
        0x8: false, 0x9: false, 0xA: false, 0xB: false,
        0xC: false, 0xD: false, 0xE: false, 0xF: false
    }

    public registerKeyPress(code: number) {
        this.keys[code] = true
    }
    //other stuff
}

class Chip8 {
    public registerKeyPress(code: string): void {
        if (Object.keys(this.keyboard).includes(code)) {
            this.cpu.registerKeyPress(this.keyboard[code])
        }
    }
    //other stuff
}
```
And if you remember, when we fetch a rom, we have custom key bindings
```ts
export interface gameMetadata {
    romLocation: string,
    keyBindings: { [customKey: string]: string }
}

export const getGameMetadata = (): { [key: string]: gameMetadata } => {
    return ({
        "Tetris": {
            romLocation: Tetris,
            keyBindings: {
                "ArrowLeft": "w",
                "ArrowRight": "e",
                "ArrowUp": "q",
                "ArrowDown": "a"
            }
        },
        //other games
```

Since it's just a keyboard event, in the `.tsx` file I could just call it when the event happens
```tsx
const someComponent: React.FC = () => {
    const registerKeyPress = (event: React.KeyboardEvent) => {
        if (chip8.current) {
            chip8.current.registerKeyPress(event.key)
        }
    }

    return(
        <someSubComponent registerKeyPress={registerKeyPress} />
    )
}
```

To kickstart the CPU cycles, we just need to call `setInterval()`
```ts
public start(clockSpeed: number, newKeyBindings?:{ [keyCode: string]: string } ): void {
    this.cpu.reset()
    this.setKeyboard(newKeyBindings);
    this.cycleInterval = window.setInterval(() => { this.cpu.cycle() }, clockSpeed)
    this.delayInterval = window.setInterval(() => { this.cpu.decrementTimer() }, 16)
}
``` 

#### React
As always, I don't really feel about writing about the React code because I didn't think it was very interesting. It was just kinda standard React stuff that I messed around with to get it looking the way I want it to. One thing that I would like to note is that I used [React95](https://github.com/React95/React95) components because I think it suits the A E S T H E T I C. 

### References
- [Austin Morlan's Blog](https://austinmorlan.com/posts/chip8_emulator/)
- [Cowgod's CHIP8 Reference](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM)
- [Tania Rascia's write up](https://www.taniarascia.com/writing-an-emulator-in-javascript-chip8/)