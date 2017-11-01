---
title: A Hack Emulator in Javascript
date: 2016/9/16 18:50:00
tags:
- nand2tetris
- javascript
- emulator
- hack
---
About a year ago <a href="http://www.jamesseibel.com">a friend</a> told me about a great book called <a  href="http://www.amazon.com/gp/product/0262640686/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0262640686&linkCode=as2&tag=lamflam-20&linkId=07d08d27757d3c2f451aa0a533038c05">The Elements of Computing Systems: Building a Modern Computer from First Principles</a><img src="//ir-na.amazon-adsystem.com/e/ir?t=lamflam-20&l=am2&o=1&a=0262640686" width="1" height="1" border="0" alt="" style="display: inline-block; border:none !important; margin:0px !important;" />. As a professional developer from a math instead of CS background, this book filled in the missing links in my mental model of the bridge between hardware and software and I highly recommend it.

After finishing the assembler project I thought it would be interesting to build a Hack (the name of the CPU architecture made for the book) emulator that could run in a browser. I've always been curious about how to actually implement an emulator and thought that this would be a simple one to start with. 

#### TL;DR
Below is a demo of the emulator and the source code can be found <a href="https://github.com/lamflam/hack_emulator">here</a>. For those that haven't gone through the book, you can download <a href="https://raw.githubusercontent.com/lamflam/nand2tetris/develop/projects/05/Fill.hack">Fill.hack</a> or <a href="https://raw.githubusercontent.com/lamflam/nand2tetris/develop/projects/05/Pong.hack">Pong.hack</a> and then load it below. For Pong, you may need to tweak the settings a little to make it playable.


<div id="hack_emulator" style="font-size: 14px; height: 400px; width: 600px; border: none; margin: 0 auto;" />
<script type="text/javascript" src="/script/hack_emulator/index.js"></script>

#### Components
<br />
##### The CPU
The Hack CPU instruction set (described in Chapter 4 of the book) consists of an A-instruction that loads a 15 bit address into register A and 28 different C-instructions, each with destination and jump bits that control where the output is stored and where to load the next instruction from. There are two registers, A and D, one pseudo-register M that represents the value at the address in register A, and the program counter. We initialize these values in the constructor of the CPU and add some setters/getters for ones that need to be exposed externally:

```javascript
...
    this._reg = {
            a: 0,
            d: 0,
            pc: 0
        };

        // The current instruction.
        this._ins = 0x0;
        this._clock = { ticks: 0 };
    }
...
```

Instead of hardcoding all 253 possible opcodes, we can encode the 28 different compute instructions and process the destination and jump bits separately. This gives a total of 30 opcodes (28 + A-instruction + NOOP).
```javascript
    get opCode() {
        // For C instructions, shift to grab the leftmost 10 bits, otherwise
        // use the hardcoded A instruction value.
        if (this._ins >= 0x8000) {
            return CPU.OPCODES[this._ins >> 6];
        } else {
            return CPU.OPCODES.LOAD_A;
        }
    }

    get dest() {
        // For C instructions, mask off the first 10 bits (the opcode portion)
        // as well as the last 3 bits (the jump portion). This leaves the
        // 3 bits corresponding to the 3 possible destination registers.
        // For A instructions, set the A register bit.
        if (this._ins >= 0x8000) {
            return (this._ins >> 3) & 0b111;
        } else {
            return 0b100;
        }
    }

    get jump() {
        // For C instructions, mask off all but the rightmost 3 bits. These 3
        // bits determine which comparison operator to use when deciding whether
        // or not to jump to the address specified in the A register.
        // For A instructions, never jump.
        if (this._ins >= 0x8000) {
            return this._ins & 0b111;
        } else {
            return 0b000;
        }
    }
```

Now that the instruction parsing is all set, we can simulate a single CPU tick quite easily.

```javascript
    tick() {
        // Read in the current instruction
        this._ins = this._mmu.readWordROM(this.PC);
        this._clock.ticks++;

        // Don't do anything if we are past the end of the ROM.
        if (this._ins < 0) return;

        // Dispatch the opcode to retrieve the current value
        let word = this[this.opCode]();

        // dest has three bits corresponding to the three registers A, D and M.
        // Store `word` in any destination with its bit set.
        let dest = this.dest;
        if (dest & 0b1) {
            this.M = word;
        }
        if (dest & 0b10) {
            this.D = word;
        }
        if (dest & 0b100) {
            this.A = word;
        }

        // jump has three bits corresponding to <, ==, and > operators.
        // The selected operator(s) compares the current value and 0, and
        // then we set the program counter accordingly.
        let jump = this.jump;
        if (((jump & 0b100) && word < 0) ||
            ((jump & 0b10) && word == 0) ||
            ((jump & 0b1) && word > 0)) {
                this.PC = this.A;
        } else {
            // If the jump comparison fails, just increment the counter.
            this.PC = this.PC + 1;
        }
    }
```

The only thing left is to implement each of the opcodes which are very straightforward. 

```javascript
...
    // this.D and this.A are setters for this._reg.a and this._reg.d
    ADD_1D() {
        return this.D + 1;
    }

    ADD_1A() {
        return this.A + 1;
    }
...
```
<br />
##### MMU
The MMU controls writing and reading memory. The Hack computer has a ROM and RAM each consisting of 32K 16-bit addressable slots. The ROM is readonly except for loading a new program. The RAM consists of normal memory as well as memory maps for the screen (`0x4000 - 0x5FFF`) and the keyboard (`0x6000`). Addresses `0x6001 - 0x7FFF` are normal RAM again.

```javascript
export class MMU {
    ...

    load(code) {
        // Code should be a string where each line contains
        // 16 1s and 0s representing the instruction.
        this._rom = [];
        code.split('\n').forEach((ins, i) => {
            this._rom[i] = parseInt(ins, 2);
        });
    }

    ...

    writeWord(addr, word) {
        if (addr < 0x4000 || (addr > 0x6000 && addr < 0x8000)) {
            return this._ram[addr] = word;
        } else if (addr < 0x6000) {
            // Adjust the address because the gPU doesn't know about the mapped
            // address range.
            this._gpu.writeWord(addr - 0x4000, word);
        } else if (addr == 0x6000) {
            return this._kb.writeWord(word);
        } else {
            throw new Error(`Memory Access Violation: ${addr}`);
        }
    }
    ...
}
```

<br />
##### GPU
There is no real GPU in the Hack computer, but this module handles writing the data from the memory map to the screen. The Hack screen is 512 pixels wide by 256 pixels high where each pixel is represented by a single bit. We represent the screen with a `canvas` and use an `ImageData` object to address and update each pixel.

```javascript
export class GPU {

    constructor(canvas) {
        this._ram = [];
        if (canvas) {
            canvas.height = GPU.rows;
            canvas.width = GPU.cols;
            this._ctx = canvas.getContext("2d");
            this._img = this._ctx.createImageData(GPU.cols, GPU.rows);
            this.reset();
        }
    }

    ...

    writeWord(addr, word) {
        this._ram[addr] = word;
        if (this._ctx) {
            // Each addr consistents of 16 pixels each represented as 4 elements
            // in the array, so the starting offset is addr * 16 * 4.
            let base = addr * 16 * 4;

            // Check each bit of this value and set the corresponding pixel
            for (let i = 0, mask = 1; i < 16; i++, mask << 1) {
                let val = (word & mask) ? 0 : 255;
                let o = base + (i * 4);
                this._img.data[o] = val;
                this._img.data[o+1] = val;
                this._img.data[o+2] = val;
                this._img.data[o+3] = 255;
            }
        }
    }
    ...
}
```

<br />
##### Keyboard

The keyboard is a very simple memory map of a single byte at address 0x6000. There are a few non-standard key mappings defined that we can easily translate to their corresponding values.

```javascript
export class KB {

    get map() {
        // Key table described in Chapter 4
        // Maps the keyCode used by the browser to the values expected by
        // the hack CPU.
        return {
            13: 128,
            8:  129,
            37: 130,
            38: 131,
            39: 132,
            40: 133,
            36: 134,
            35: 135,
            33: 136,
            24: 137,
            45: 138,
            46: 139,
            27: 140,
            112: 141,
            113: 142,
            114: 143,
            115: 144,
            116: 145,
            117: 146,
            118: 147,
            119: 148,
            120: 149,
            121: 150,
            122: 151,
            123: 152
        }
    }
    ...
    writeWord(word) {
        return this._key = this.map[word] || word;
    }
}
```

<br />
#### The Emulator
##### Putting it all together
Lastly, we need to hook the components together so that the Hack computer runs and renders to the screen. After trying a few different rendering methods, I decided to do a full screen render at a user defined refresh rate using `requestAnimationFrame` because it gave better results than rendering each pixel on write.

```javascript
    render(ms) {
        // Rendering on the full canvas on every frame could be expensive,
        // so only render every refreshMS milliseconds.
        if ((ms - this._lastRender) > this._refreshMS) {
            this._gpu.render();
            this._lastRender = ms;
        }
        // Render again
        if (!this._stop) requestAnimationFrame(this._renderer);
    }
```

Using a `setTimeout` per CPU tick is WAY too long even with a timeout of 0. Instead in each `setTimeout` call we run a configurable number of ticks so that we can control the CPU speed a little better since Hack computer doesn't have a real timing mechanism.

```javascript
    step(force) {
        // Run through this._ticks CPU cycles. We need to stop looping and
        // use setTimeout every once in a while to make sure that our event
        // handlers have a chance to run to register keypresses, etc.
        for (let i = 0; i < this._ticks; i++) {
            if (!this._stop || force) this._cpu.tick();
        }
        if (force) {
            this.render(this._lastRender + this._refreshMS + 1);
        } else {
            this._timer = setTimeout(this._stepper, 0);
        }
    }
```
