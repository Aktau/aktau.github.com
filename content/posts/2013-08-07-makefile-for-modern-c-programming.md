---
title: A makefile for modern C programming on UNIX-like operating systems
created_at: 2013-08-07
description: Presents a (hopefully) understandable makefile that can be used for cross-platform, cross-compiler projects
kind: article
tags: [unix, c, make, linux, osx]
---
Looking for an easy to use build tool? Project not yet large enough to warrant cmake? Downright scared of autotools (which generates the scariest kind of makefiles)? May I present... make!

<!-- more -->

Most of you have probably seen what a makefile looks like, either from another open source project or automatically generated by tools like [Dev-C++](http://orwelldevcpp.blogspot.de/) [^1] or (god forbid) autotools. They usually have one thing in common: **they're horrendous, unreadable messes**. And it makes you never want to touch make with a 10-foot pole.

It was just recently that I learned that it didn't have to be that way, make can be small and simple. Let's start with the simplest of makefiles, which compiles a single .c file into an application. Save the following into a file with the name Makefile, next to a main.c file. [^2]

~~~~~~~~
#!make
myapp: main.c
    gcc -o myapp main.c -I.
~~~~~~~~

To run it, execute:

~~~~~~~~
#!bash
$ make myapp
# or... absent a specific target, make will
# just execute the first one it finds.
$ make
~~~~~~~~

(if you get strange errors when you try this, it's because you haven't been indenting your lines with a TAB-character. Make explicitly requires that lines be indented with tabs or it will throw a hissy fit and spout poorly worded error messages from which it is impossible to infer that it actually wants tabs.)

So, what does this do? Make will try to run gcc in the following cases:

- the file *myapp* DOES NOT exist and all dependencies are present
- the file *myapp* DOES exist and all dependencies are present, but one of the dependencies has a newer modification date than myapp

If one of the dependencies is missing and make doesn't know how to make it (with another rule), make will error out.
Likewise if one of the steps produces an error, make will stop (unless specifically told to ignore the error).

Put more abstractly, make is just trying to satisfy its rules, which look like this:

~~~~~~~~
#!make
<output>: <dependencies>
    <steps to make output from dependencies>
~~~~~~~~

So, in these terms, make's reasoning becomes clearer: to get **output**, I need **dependencies** and then I need to run **steps to make output from dependencies**. If the output already exists, make will do nothing. If the dependencies are lacking, make will try to make them if it has a rule for them. Please note that the part called output is often also called a target.

Many people will start protesting now, as they've seen "outputs" like **install, uninstall, clean, etc.** that don't generate files called install, uninstall or clean. The thing is that these targets are special, and they are usually indicated as such by a special target called **.PHONY**. Make doesn't need or want to know about the files generated by phony rules. Phony rules will always execute when called, multiple invocations to `make install` will, by default, do the same thing.

In the case above, *main.c* already exists, so it doesn't need to be made. Lucky for us,
as we didn't specify a rule to make *main.c*.

In most C projects, there's a tendency to first generate the object (.o) files and then
generate the application from them. Doing this presents a nice opportunity to show multiple
rules working in tandem in make:

~~~~~~~~
#!make
myapp: main.o
    gcc main.o -o myapp -I.

main.o: main.c
    gcc -c main.c -o main.o
~~~~~~~~

So now we're first compiling to object files and then linking them together into
an executable, great! But what if we want to add another file? Suppose we have
another file called helper.c that we want to compile and link into our executable,
we could do this:

~~~~~~~~
#!make
myapp: main.o helper.o
    gcc main.o helper.o -o myapp -I.

main.o: main.c
    gcc -c main.c -o main.o

helper.o: helper.c
    gcc -c helper.c -o helper.o
~~~~~~~~

Note that we added helper.o as a new dependency for myapp, and that we specified
a rule for how to build helper.o from helper.c.

This works perfectly fine, but it's getting kind of repetitive. Is there no way we
can fold the two last rules into one? Basically the only thing that differs
between them is the filename.

Sure, but that's usually where it gets hairy for someone not accustomed to make.
Make has some special variables you can use inside of a rules to get rid of
the redundancy, but they're very (very) poorly named.
The following four are pretty important, for starters:

- **$@**: the name of the target file (the one before the colon)
- **$<**: the name of the first (or only) dependency (the first one after the colon)
- **$^**: the names of all the depdencies (space separated)
- **$***: the stem (the bit which matches the % wildcard in a rule definition. (I'm not using this now, but it could be handy someday)

These special variables, combined with wildcards (the **%** symbol in make), allow us
to compactly eliminate all the redundancy. An example will probably clarify it
better than a 1000 words.

~~~~~~~~
#!make
myapp: main.o helper.o
    gcc $^ -o $@

# when looking for something that ends in .o, look
# for the same thing ending in .c and run gcc on it
%.o: %.c
    gcc -c $<
~~~~~~~~

There, redundancy solved! We only had to specify the name of the executable, and the object files
that are necessary to build the executable exactly once. If you expand
the variables in your head, it also looks quite logical.

Now there are some tiny tweaks that I do quite often to add
some commandline overridability:

~~~~~~~~
#!make

# if $CC is not set, use gcc as a sensible default
CC ?= gcc

# if $CFLAGS is not set, be very pedantic and compile
# as C11, that should catch some common errors, also
# fortify the source, which is a must for security.
CFLAGS ?= -Wall \
    -D_FORTIFY_SOURCE=2 \
    -Wextra -Wcast-align -Wcast-qual -Wpointer-arith \
    -Waggregate-return -Wunreachable-code -Wfloat-equal \
    -Wformat=2 -Wredundant-decls -Wundef \
    -Wdisabled-optimization -Wshadow -Wmissing-braces \
    -Wstrict-aliasing=2 -Wstrict-overflow=5 -Wconversion \
    -Wno-unused-parameter \
    -pedantic -std=c11

myapp: main.o helper.o
    $(CC) $^ -o $@ $(CFLAGS)

# when looking for something that ends in .o, look
# for the same thing ending in .c and run gcc on it
%.o: %.c
    $(CC) -c $< $(CFLAGS)
~~~~~~~~

That cranks the warnings up to 11, which is often
a good thing. It's a good idea to turn the warnings on when
you start your project. Solving the deluge of warnings that
can come out of a mature project when going from no flags
to very pedantic is not fun.

Of course, there are some flags
that might be a little bit too pedantic. For example, when
doing game development it's often useful to do some
[bit twiddling](http://graphics.stanford.edu/~seander/bithacks.html),
compare disparate number types and do dirty things with pointers.
In that case it might not be worth it or even possible to
prevent all the warnings by casting to the appropriate type. In
that case, feel free to disable some flags.

### Debug and release builds

Quite often, you'd want to compile in debug mode but be
able to run

~~~~~~~~
#!bash
$ make release
~~~~~~~~

When you're done, spitting out a fully optimized and stripped
executable.

In make, there are often quite a few ways to achieve the
same thing, adding to the confusing. For debug and release
builds, I personally went for something really simple, expanding
our last example:

~~~~~~~~
#!make
# if $CC is not set, use gcc as a sensible default
CC ?= gcc

# if $CFLAGS is not set, be very pedantic and compile
# as C11, that should catch some common errors, also
# fortify the source, which is a must for security.
CFLAGS ?= -Wall \
    -D_FORTIFY_SOURCE=2 \
    -Wextra -Wcast-align -Wcast-qual -Wpointer-arith \
    -Waggregate-return -Wunreachable-code -Wfloat-equal \
    -Wformat=2 -Wredundant-decls -Wundef \
    -Wdisabled-optimization -Wshadow -Wmissing-braces \
    -Wstrict-aliasing=2 -Wstrict-overflow=5 -Wconversion \
    -Wno-unused-parameter \
    -pedantic -std=c11

CFLAGS_DEBUG := -g3 \
    -O \
    -DDEBUG

CFLAGS_RELEASE := -O2 \
    -march=native \
    -mtune=native \
    -ftree-vectorize

# the default target is debug
all: debug

# when the target is debug,
# add CFLAGS_DEBUG to CFLAGS
debug: CFLAGS += $(CFLAGS_DEBUG)
debug: myapp

# when the target is release,
# add CFLAGS_RELEASE to CFLAGS
release: CFLAGS += $(CFLAGS_RELEASE)
release: myapp

myapp: main.o helper.o
    $(CC) $^ -o $@ $(CFLAGS)

# when looking for something that ends in .o, look
# for the same thing ending in .c and run gcc on it
%.o: %.c
    $(CC) -c $< $(CFLAGS)

.PHONY: debug release
~~~~~~~~

We're using **target-specific variables** to get
the job done. Notice that we added release and debug
as phony targets because they don't generate files
called release and debug. Also note that both
target and debug have myapp as a dependency, so
they will both build the executable we want,
albeit with different flags.

### Taking into account differences in operating systems or compilers

So now you've got your fancy project building in debug and release modes
and you're really happy about it, but what when you've been developing
on OSX and want to build &amp; run it on Linux as well? Or what if
you want to support clang because of it's awesome diagnostics?

With make, you can run some commands to find out what your environment
looks like and make choices based on that. The long and short of it
can be found on [stack overflow](http://stackoverflow.com/questions/714100/os-detecting-makefile).
I'll repaste my own edited version here for posterity:

~~~~~~~~
#!make
ifeq ($(OS),Windows_NT)
    CCFLAGS += -D WIN32
    ifeq ($(PROCESSOR_ARCHITECTURE),AMD64)
        CCFLAGS += -D AMD64
    endif
    ifeq ($(PROCESSOR_ARCHITECTURE),x86)
        CCFLAGS += -D IA32
    endif
else
    # tries to find the compiler name
    CC_VERSION := $(shell $(CC) --version | head -1 | cut -f1 -d' ')

    # tries to discern what UNIX-like OS we're running on
    UNAME_S := $(shell uname -s)

    ifeq ($(UNAME_S),Linux)
        CCFLAGS += -D LINUX
    endif
    ifeq ($(UNAME_S),Darwin)
        CCFLAGS += -D OSX
    endif

    ifneq (,$(findstring clang,$(CC_VERSION)))
        CCFLAGS += -D CLANG

        # -pthread is not necessary when using Clang on Darwin
        ifneq ($(UNAME_S),Darwin)
            CCFLAGS += -pthread
        endif
    else
        CCFLAGS += -D GCC
        CCFLAGS += -pthread

        # at least on OS X 10.7.5, the apple linker does not understand AVX, and gcc uses it natively
        ifeq ($(UNAME_S),Darwin)
            CCFLAGS += -mno-avx
        endif
    endif

    UNAME_P := $(shell uname -m)

    ifeq ($(UNAME_P),x86_64)
        CCFLAGS += -D AMD64
    endif
    ifneq ($(filter %86,$(UNAME_P)),)
        CCFLAGS += -D IA32
    endif
    ifneq ($(filter arm%,$(UNAME_P)),)
        CCFLAGS += -D ARM
    endif
endif
~~~~~~~~

It adds define flags so that whenever necessary, it can be used
to partially define blocks of code based on OS, CPU or compiler.
Of course I should note that this is best used when there are no
obvious alternatives, nobody likes `#ifdef` soup.

By the way, I was bashing autotools earlier for being a mess and creating unreadable
makefiles, but it remains an oft-used toolset for cross-platform building.
Why is that? Inertia? Well... yes and no. Sometimes building on many (very) different
operating systems becomes quite a chore, and autotools makes some of that, well,
easier. It was built long ago for the express purpose of generating cross-platform
makefiles, which is also why it panders to the lowest common denominator by
not using any of the newer features that modern make has.

There are [a lot](http://cgit.freedesktop.org/libva/) of
[examples](http://cgit.freedesktop.org/xorg/driver/xf86-video-intel/) of
[projects](http://www.mplayerhq.hu/design7/news.html)
using autotools to great effect, even maintaining almost readable autoconf.ac files.
One could copy and paste given a bit of effort and after a while you could be an
autotools adept too (not a wizard, I suppose there are only 3 people in the world like that).

A decent alternative is [cmake](http://www.cmake.org/), which tends to be a bit more readable, and can generate
makefiles on UNIX platforms and visual studio project files on windows, if that's
your thing.

But when your project exhibits just slight differences between OS or compiler toolchains,
there's no need to take the plunge and migrate to autotools, cmake or anything else just yet.
We can make do with plain make, and still keep it quite readable.

[^1]: Sweet nostalgia, Dev-C++ was my first real IDE, it helped me form my knowledge of C by being easy to use yet not having auto-completion, which cemented a lot of important functions in my muscle memory.

[^2]: Note that I'm describing makefiles for a C project here, but any developer worth her salt should be able to see that it's applicable to much more. In fact I use make in combination with [fpm](https://github.com/jordansissel/fpm) to build .deb files for debian/ubuntu, all I have to do is run `make deb`.