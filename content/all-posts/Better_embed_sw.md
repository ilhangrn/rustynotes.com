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




