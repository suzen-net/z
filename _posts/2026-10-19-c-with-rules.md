---
layout: post
title: "Writing C with borrowed rules, and what Zig does instead"
---

I am kind of into bare-metal servers these days, the other end of the
spectrum from container orchestration and lambdas, which are where most
of the attention is. That got me thinking about using resources
efficiently, which is not always the first thing on people's minds when
a platform scales almost without limit. I needed something close to real
time, with durability guarantees, that could live on very little memory,
because the same machines host plenty of other things besides it. That
is how msgsrvd was born.

[msgsrvd](https://github.com/cienvita/msgsrvd) is a message log server I
am writing at Cienvita. It runs on bare-metal Linux, every acknowledged
write has to be on disk, and it is written in C99 with no libc: raw
syscalls, no `malloc`, and statically linked.

Zig would have been the obvious choice for this today. I picked C partly
for the challenge, and partly because I wanted to see what the things I
take for granted in higher level languages, error handling above all,
actually cost when you have to build them yourself. Doing that also made
me appreciate how much Zig gives you in the language itself.

I also wanted predictable performance without GC pauses, which ruled out
C#, the language I know best (yes, I know: Rust).

C is notoriously dangerous, but I thought I could tame it by dealing
with two of the main sources of bugs: memory management and error
handling. None of the rules I followed are mine. These are the people I
took them from.

## Arenas instead of malloc and free

Ryan Fleury, [Untangling Lifetimes: The Arena
Allocator](https://rfleury.com/p/untangling-lifetimes-the-arena-allocator)
(2022), and Chris Wellons, [Arena allocator tips and
tricks](https://nullprogram.com/blog/2023/09/27/) (2023).

The idea is to group allocations by lifetime and free the whole group at
once. In msgsrvd each connection gets an arena over a fixed buffer.
Allocation is a bump of a watermark, there is no individual free, and
when the connection is done the arena is reset:

```c
typedef struct {
    uint8_t    *base;       /* backing buffer (only raw pointer we hold) */
    int32_t     capacity;
    int32_t     used;
} arena_t;

static inline void arena_reset(arena_t *a) { a->used = 0; }
```

Most of the questions `free` makes you answer in every function go away,
because the answer is always "when the connection closes".

## Indices instead of pointers

Andre Weissflog, [Handles are the better
pointers](https://floooh.github.io/2018/06/17/handles-vs-pointers.html)
(2018).

His post is about handing out array indices instead of pointers, so the
owner of the memory keeps control of it. msgsrvd takes a narrow version
of that: an allocation is a span, an offset and a length into a known
buffer, and becomes a pointer only at the point of use, with a bounds
check:

```c
typedef struct {
    int32_t     off;
    int32_t     len;
} span_t;

static inline uint8_t *span_ptr(uint8_t *base, int32_t base_len, span_t s)
{
    if (s.off < 0 || s.len < 0 || s.off + s.len > base_len)
        return NULL;
    return base + s.off;
}
```

A span is eight bytes, it survives the buffer being moved, and it can be
written to disk or sent to a replica as it is.

## Your own types, signed sizes, no libc

Chris Wellons, [My personal C coding style as of late
2023](https://nullprogram.com/blog/2023/10/08/).

Wellons defines his own primitive types, uses signed sizes, and replaces
null-terminated strings with a pointer and a length. msgsrvd goes one
step further and drops libc entirely. Fixed-width types are declared
locally, lengths are `int32_t`, and every syscall is a few lines of
inline asm:

```c
static inline long sys_call1(long n, long a1)
{
    long ret;
    __asm__ volatile("syscall"
        : "=a"(ret)
        : "a"(n), "D"(a1)
        : "rcx", "r11", "memory");
    return ret;
}
```

The practical payoff is the build: the binary links statically, takes
nothing from libc but the startup code, and runs on the servers without
a glibc version having to match the machine that built it.

## No allocation after startup

Gerard Holzmann, [The Power of Ten: Rules for Developing Safety-Critical
Code](https://en.wikipedia.org/wiki/The_Power_of_10:_Rules_for_Developing_Safety-Critical_Code),
IEEE Computer, 2006, and TigerBeetle's
[TigerStyle](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md),
which cites it.

Holzmann wrote ten rules for flight software at JPL: bounded loops, no
heap allocation after initialization, check every return value, compile
with every warning on. TigerStyle puts the allocation rule as "All memory
must be statically allocated at startup." msgsrvd follows it: all memory
is reserved up front, a segment file is allocated to its full size when
it is created so appends never extend it, and the build runs with
`-Wall -Wextra -Wpedantic -Werror`.

The result is a server whose memory use is fixed by its configuration.
Under load it refuses work rather than growing.

## Errors as values, by hand

This is the part I most wanted to try. There are no exceptions in C and
I did not want `errno` and `-1`. Every fallible call returns a small
`result_t`, and an `err_t` is passed down the stack. When something
fails, each layer pushes a frame with its file, function, line and some
structured detail, so the debug build can print where the error started
and how it travelled:

```c
r = os_openat(e, dir_fd, name,
              O_RDWR | O_CREAT | O_EXCL | O_CLOEXEC, MODE_0600, &fd);
if (!result_ok(r)) {
    ERR_PUSH(e, ERR_STORAGE);
    return r;
}

/* Allocate the whole segment up front so appends never extend it. */
r = os_fallocate(e, fd, 0, 0, (uint64_t)capacity);
if (!result_ok(r)) {
    ERR_PUSH(e, ERR_STORAGE);
    os_close(e, fd);
    return r;
}
```

The error stack has a fixed number of frames. When it fills up it keeps
the frames where the error started and the most recent callers, and
counts the ones it dropped in between.

It works, and it is a lot of ceremony. Every call site is four lines,
cleanup on failure is repeated by hand, and nothing stops you forgetting
to check `r`. That cost is the thing I came to C to feel.

## Where Zig lands

Zig's [overview](https://ziglang.org/learn/overview/) lists "no hidden
control flow" and "no hidden memory allocations", and says errors are
values that may not be ignored. Most of what msgsrvd does by convention,
Zig does in the language:

| msgsrvd, by hand | Zig |
| --- | --- |
| `span_t` and `span_ptr()` | slices carry a length and are bounds-checked in safe builds |
| an `arena_t` passed around | functions that allocate take an allocator; the standard library has an arena |
| `result_t`, `if (!result_ok(r))` | error unions and `try` |
| `ERR_PUSH` frames | [error return traces](https://ziglang.org/documentation/master/#Error-Return-Traces) |
| repeated cleanup on each failure path | `errdefer` |
| `STATIC_ASSERT` with a negative array size | `comptime` |
| own types, inline asm syscalls, `-static` | no libc needed on Linux, static builds by default |

The segment code above comes out roughly like this, leaving out the
exact standard library calls:

```zig
const fd = try openSegment(dir_fd, name);
errdefer close(fd);
try preallocate(fd, capacity);
```

It keeps the parts of C I was imposing on myself, explicit allocation
and errors as plain values, and makes the compiler enforce them. What it
does not do is hide anything, which is the line between it and the
languages with exceptions and a garbage collector.

## Would I pick C again

For this project, yes. I don't want to deal with concepts I can't see
through, or with consequences I only find out about the hard way. In C
everything the server does is in front of me, including the parts a
compiler could have done for me, and writing and reviewing those myself,
with a lot of help from Claude Code, is how I learned what they cost.

I used Claude Code, Anthropic's AI coding tool, heavily on this
project, mostly with the Claude Opus model.
