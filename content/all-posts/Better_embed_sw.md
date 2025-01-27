+++
title = "Better Embedded Software"
date = 2025-01-27
[taxonomies]
categories=["Guides"]
tags=["Embedded tech", "Software", "Source", "Guide"]
+++
---
<br>

## (*Eng*) Notes from video: Better embed sw
> Most of time we see header files like:

``` c
#define TMOD    ((unsigned volatile *)0X3FF6000)
#define TDATA   ((unsigned volatile *)0X3FF56004)

// but this is not proper way. size varies in different platforms.

// instead we use
typedef uint32_t volatile dev_reg;

// then we see them like this
#define TE      0x1 //bit_mask
#define TMOD    ((dev_reg *)0x3FF6000)
#define TDATA   ((dev_reg *)0X3FF6004)

// UART0 registers
#define ULCON0  ((dev_reg *)0x3FFD000)
#define UCON0   ((dev_reg *)0x3FFD004)

// UART1 registers
#define ULCON1  ((dev_reg *)0x3FFE000)
#define UCON1   ((dev_reg *)0x3FFE004)
```

> Example use case of them would be like:
``` c
*TMOD |= TE;    // set the timer enable bit
*UTXBUF0 = c;   // write c's value to UART0
```

> And it is error-prone, we do mistakes almost in every project
``` c
void (UART_put)(dev_reg *stat, dev_reg *txbuf, int c);
// here we pass status register, buffer register and count
// but it is error prone, because we can mix them

UART_put(UTXBUF0, USTAT0, c);   // wrong order
UART_put(USTAT0, UTXBUF1, c);   // mismatching UART #s
UART_put(TMOD, UTXBUF1, c);     // wrong device

// They all will be compiled but will hvae to be debugged.
```

> He is Dan Saks, saying people have frames in their brains. Any idea doesn't fit will just bounce, regardless what you say and what you know.

> Lets improve the code and give compiler time errors instead of hours of debugging.
``` c
// lets wrap them in struct
struct timer {
    dev_reg TMOD;
    dev_reg TDATA;
    dev_reg TCNT;
};

void timer_enable(timer_t *t);
uint32_t timer_get(timer_t *t);

```

> Lets do it for UART
```c
// this is getting popular among companies, the right way.
struct UART{
    dev_reg ULCON;
    dev_reg UCON;
    dev_reg USTAT;
    dev_reg UTXBUF;
    dev_reg URXBUF;
    dev_reg UBRDIV;
};

void UART_put(UART *u, int c);
int UART_get(UART *u);

UART *const com0 = (UART *) 0x3FFD000;
timer *const timer0 = (timer *) 0x3FF6000;

UART_put (com0, c); // put c to a UART object
UART_put(timer0, c); // compiler error! but just in c++
// in c compiler, it may might issue a warning, but doesn't have to
```

> The other problem we have in embedded ecosystem is compiler. Vendors don't update it, most of them uses ==c99==, not c11. Even for c++ they support just c++03.
- It is not profitable to create compiler for a mcu, pc side is more profitable there are millions of products.
- And ecosystem is fine with old one, they are comfortable and don't ask for new one. Why should they move, change everything, all system?
- Mentions Daniel Cohneman's book and theory. People has outsized fear relateive to what they might lose and what they might gain. 

It is like:

    Fear of loss == 2 * (desire for gain)

And one more great writer comes with idea which matches:

    So the only way .. to influce other poeple is to talk about what they want and show them how to get it.

Another key note:

    Make interface easy to use correctly and hard to use incorrectly

> C has static data types. You declare it with it's type. Which is the way it is for the entire duration of execution of program. What it data type: it is a bundle of compile-time properties for an object.
- size and alignment
- set of valid values
- set of permitted operations

> Accumalated wisdom injected in language and set the boundaries. For example an integer is not allowed
- *i    // indirection(as if a pointer)
- i.m   // member selection
- i()   // call(as if a function)

> Here are some implicit type convertions.

```c
int i;
long int li;
double d;
char *p;

li = i; // convert int into long int, ok in c
d = i;  // convert int into double, ok in c
d = p;  // error, convertion pointer into double

// C++ will reject all of them at compile time
```

> In c++ their a modern approach which set the bar too high
- Use streams instead of FILEs
- Use vectors instead of arrays
- Use string instead of a null-terminated character sequences.

> But this approach is not helpful for C developers. C++ was supposed to be "Better C". Part of it would not be deficit, it should be compability with C.

> There is a padding issue with structures. Registers are usually 32bits, and when we create structures with 8bit 16bit values, compiler applies padding to organize. To be sure that our structure keeps registers in correct places we apply static_assert like:
```c
struct UART{
    dev_reg ULCON;
    dev_reg UCON;
};

static_assert(
    offsetof(UART, UCON) == 4,
    "UCON member of UART is at the wrong offset"
); // this will cause failure if there is something else in the place you except your data to be.

// if it is a big structure we don't need to check every part, instead we check sizeof it like
static_assert(
    sizeof(UART) == 6 * sizeof(dev_reg),
    "UART contains extra padding bytes"
);
```

> Let's hit the goal. Instead of this struggle with structs, why don't we use classes. Classes will keep the register in a memory which are not directly accessible. Now you can limit access.

> Classes are not anything more magical than structs with constrained operations. It is better way of doing compile time type checking. Here it is 
```c++
class UART {
    public:
        void put(int c);
        int get();
    private:
        dev_reg ULCON;
        dev_reg UCON;
};

com0-> put(c);
```

> What about the cost of using class?

    Zero, Zip, Zilch, Nothing, Nil, Nada.
    
    Same size, same speed
    Sometimes even faster

> And also if you don't like you can use your c style

    com0->put(c);       // C++
    UART_put(com0, c);  // equivalent C

> Lets look interrupts. They are void functions without parameters. And need to be placed in IVT-interrupt vector table
```c
// lets say adr in our vector is 0x38, we want to add our function
*(void **) 0x38 = (void *) IRQ_handler;

// and our function is
void IRQ_handler();

// But we have issues, it is crypic, hard to understand.
// (void **) is casting 0x38 to pointer to pointer. So it will point the address of function. Then with first * it places there a function pointer for IRQ.

// but there is a concern in assigning function pointer to pointer to pointer. They are not same type and who knows is there any bit lose. Undefined behaviour.

// right side is pointer to function
// left side is pointer to data

```

> In C++ there is a solution for this:
```c++
typedef void (*ptr_to_handler)();   // c++03 or c++11
using ptr_to_handler = void(*)();   // c++11

*(void **)0x38 = (void *)IRQ_handler;   // before
*(ptr_to_handler *) 0x38 = IRQ_handler; // after

// I think i have seen second one several times. It is not just for C++, we can do it in C
```

> lets check a better way
```c
enum interrupt_number{
    reset,
    undefined_instruction,
    SWI,
    prefetch_abort,
    data_abort,
    reserved,
    IRQ,            // plain device interrupts
    FIQ             // fast device interrupts
};

// here it is function pointer clearly
typedef void (*ptr_to_handler)();   // c++03 or c++11

// here we IVT shows function pointers on 0x20
ptr_to_handler *const IVT = (ptr_to_handler *)0x20;

// in modern c++, just to see here.
auto const IVT = reinterpret_cast<ptr_to_handler *>(0x20);

// then the call becomes
IVT[IRQ] = IRQ_handler; // ok in C and C++

```

> And yes, our weird all open coding helps us on debugging but the purpose is is getting errors in compiler time, not in debug.
```c
// most is like this
if((48 < c) && (c <= 57))   // is c digit

// this is better
if(('0' <= c) && ( c <= '9'))    // is c digit

// but we may have
if(isdigit(c))       // probably faster, too
```

> For our IVT example, it is still not perfect.

    we may have IVT[42] = IRQ_handler; no boundary check, it is not RUST

> But we can turn it into class.
```c++
class IVT{
    public:
        using pointer = void(*)();
        enum number{                // was interrupt_number
            begin, reset = begin,...,IRQ,IFQ,end
        }; // this a strange way of enums, i didn't know.
        // but is seems smart, i need test in C 

        pointer &operator[](number n){ // why do we have &[] here? is not it a function ?
            return table[n];
        }
    private:
        pointer table[end-begin];
};

// then it becomes
IVT &the_ivt = *reinterpret_cast<IVT *>(0x20);

the_ivt[IVT::IRQ] = IRQ_handler;
```

> But using these C++ features and skills we can catch the bugs in compiler time instead of run-time(debugging). If you are an embedded c developer and want to migrate to c++, forget the generics and templates. Focus on what you are going use most and always in front.
- enumerations
- (lvalue) reference types
- const and constexpr
- function and operator overloading
- classes as structure with(constrained behaviour and guaranteed initialization and descruction)
