# Bare-Bones Program #3

* **Code file: [bbones3.asm](./bbones3.asm "Link to source code file for bbones3.asm")**
* [Run in-browser](https://8bitworkshop.com/v3.3.0/embed.html?p=vcs&r=TFpHAAAQAAAAAGV%2FXxT7AQECAwR42KL%2FmqIAoACUAOiKyf%2FQ%2BIQAqSqFCUwT8P8EHwQfBB8EHwQfBB8EHwQfBB8EHwQfBB8EHwQfBB8EHwQfBB8EHwQfBB8EHwQfBB8EHwQfBB8EHwQfBB8EHwQeBBcA8ADw "Link to in-browser emulation of bbones3.asm") at 8bitworkshop
  
  
Confession time: We haven’t been being good 2600 citizens so far. There’s a certain amount of setup we really should be doing at the beginning of our program. Like a trained chef who gets his or her [*mise en place*](https://en.wikipedia.org/wiki/Mise_en_place "Wikipedia article on mise en place") arranged before cooking a meal, we have to prepare our virtual working area and make sure everything is in its place.

On an emulator, this isn’t super-important. Typically, the microprocessor and the entire memory map start as a blank slate. But on a real 2600, those registers and addresses could be set to who-knows-what. That can cause trouble down the line. We need to take care of things by doing this:

## Typical Initialization Steps

1. Make sure the chip is ignoring interrupts (even though it probably won’t get any, as discussed earlier).
1. Ensure that the processor is in the normal, binary math mode.
   * The 6502 family can operate in a special "Binary-Coded Decimal" mode that can be quite useful in certain situations.
   * This is not one of those situations. So we *really* want to make sure that BCD mode is off if we want addition and subtraction to behave as expected.
1. Set the “stack pointer” to its highest value. We’ll talk more about this later, but for now, these are the two things to know:
   * The stack pointer marks the *bottom* of a special section of RAM called the "stack". This location can vary as the stack changes in size.
   * Nearly every program you write will **also** be using RAM for storing what are essentially variables.
   * Since we don't want our variables area and the stack area to stomp all over each other, we'd prefer that the stack start out as far out of our way as possible.
1. Set every register in the TIA chip to zero.
1. Set every byte in RAM to zero.

## Basic Set Up

The first part of our program takes care of steps 1-3:

```assembly
; Basic set-up

Start   sei         ; Prevent interrupts
        cld         ; Clear "binary-coded decimal" mode
        ldx #$FF    ; Take the highest address in RAM...
        txs         ; ...and put it in the stack pointer
```

The first two instructions are pretty straightforward. They're both single-byte codes that do one specific task and don't require any value to follow them--akin to a function that takes no arguments:

   * `sei` - **SE**t **I**nterrupt disable
   * `cld` - **CL**ear binary-coded **D**ecimal mode

The next two lines set the stack pointer. In the previous two versions, we used the microprocessor’s A register (aka the Accumulator) as our “clipboard” to transfer a value to a specific place. Here we’re doing basically the same thing, but we're using a different register on the chip: The X register. The instruction itself is almost identical, only it's `ldx` (**L**oa**D** **X**) instead of `lda`.

The X register (along with its sibling, the Y register) is not as full-featured as the primary A register. But you can put things into it, take things out, and increment or decrement it. It works great as an index or counter register, which is what you'll mostly use it for.

But one thing X can do that that A cannot is transfer its contents to the stack pointer (via `txs`: **T**ransfer **X** to the **S**tack pointer). That's why we're using X here instead of A... we don't really have any other choice!

Also notice that we’re not “storing” X in the stack pointer, but rather “transferring” it. And we don't need to specify an address. That’s because the stack pointer is not at an external location--it's a register living right on the microprocessor, just like A, X, and Y. Moving data between local microprocessor registers is, in the jargon of the 6502 instruction set, a *transfer* rather than a *store*.

## Initializing the "Zero-Page"

Due to the way the VCS maps all of its address locations, the various TIA registers, RAM areas, etc. that we need to clear out all happen to be found in the first 256 bytes of address space, from $0000 to $00FF. Lucky for us, this "zero-page" area can be handled very efficiently by the 6502 family.

> Fun Fact: A *page* is any of the contiguous, 256-byte blocks that divide up the 6502's available address space.

For our first stab at this, we'll use an approach similar to what you're probably used to in a high-level langauage. This is **not** the most efficient way to do this. (Spoiler alert: We'll rewrite it in the next example. Bear with me until then!)

The basic steps are sort of like a FOR loop or a WHILE loop:

1. Keep track of the current address location, starting at address zero.
2. Write a zero to wherever our current address location is.
3. Increment our current address to the next location.
4. Have we gotten to the top (i.e., address $FF)?
   * If so, exit the loop
   * If not, go to step 2

Note that the A, X, and Y registers are each only 8-bits in size. So $FF is the largest value they can hold. If you try to bump them up one higher, they'll just roll back over to zero rather than wind up at $100.

So for now, we'll stop our loop when our target address *equals* $FF rather than *exceeds* it. As you might have already figured out, this will leave the final byte uninitialized by the loop, so we'll need one more instruction at the end to fix that.

The assembly-language version of these steps looks like this:

```assembly
        ldx #0 
        ldy #0
        
Init    sty 0,x

        inx

        cpx #$FF
        bne Init
        
        sty $FF
```

We're using the X register as our address pointer. The X and Y registers are designed to be *index registers*, and often used in cases like this, where we want to iterate or count over a range of values.

Y just holds the zero that we'll write to the zero-page. It's our "clipboard" this time. We could've just as easily used the A register instead. No real significance in going with Y here. Just mixing things up a bit!

At the beginning of the `Init` loop, we store the contents of Y (which we're keeping at zero) to wherever X is pointing. Well, actually we're writing it to address "zero plus X". There is no 6502 assembly instruction that simply stores a value to address X directly. But we can use X as an offset to some constant address. In this case, our constant is zero, so it's effectively a "store Y in address X" instruction when all is said and done.

Next we check to see if we need to exit the loop, depending on whether or not X has made it to $FF yet. In assembly, this sort of general-purpose "IF-THEN" operation takes two steps:

1. Compare the contents of a register to some value
1. Do something based on what the results of that comparison were

So that's what we do. The `CPX` **C**om**P**ares **X** to the hex value $FF. This doesn't change any normal register contents, but it does set some "flags" (think of them as global boolean variables) on the chip. These **status flags** keep track of whether the most-recent math operation resulted in a zero, or in a negative number, and/or caused a mathematical "carry" to occur, and a few other things like that.

Finally, we check the result of the comparison (by looking at whatever flags are now set) and act accordingly. In this case, we do a **B**ranch if the previous comparison found that the two values were **N**ot **E**qual. That keeps our loop running until X equals $FF.

To put it another way, conditional branching in assembly is similar to the "IF-THEN" process you're familiar with, except the "IF" part always involves one of the status flags. You can't *directly* branch based on, for example, what X does or doesn't equal, or is greater or less than. But you can do something to X (like a compare operation) that just so happens to set one or more of the status flags as a side-effect. You then branch based on one of those flags.

> **Note:** The comparison is actually a subtraction that throws away the result. If the values being compared (the contents of X and whatever value is being used as an argument to the `cpx` instruction) are equal, then the subtraction operation results in a zero, which flips on the zero flag. If the values are different, the subtraction will *not* result in a zero (thus turning *off* the zero flag) but might do other things such as set the negative flag or the carry flag. The `bne` instruction just has to look at the zero flag to know whether to branch or not.

We wrap up with one last store of Y into the highest address of the zero-page (since the loop would've missed it).

The code in this init routine takes up 13 bytes of ROM and take almost 3,000 cycles (about 2.5 milliseconds) to execute. Not bad--but **we can do better!** Stay tuned...



## Review

* It's (usually) a good idea to start your VCS programs by initializing the system and clearing out the TIA and RAM.
* In addition to the Accumulator, the registers on the 6502-family microprocessors include X, Y, and SP (the stack pointer). There are also several "flags" that keep track of various things.
* "Branching" is the assembly-language equivalent of a conditional ("if") statement. It relies on previous operations setting certain flags.

Next, we'll re-write that zero-page loop to a more efficient form that's more idiomatic to assembly language.

#### Next example: [bbones4.asm](./bbones4.md)
