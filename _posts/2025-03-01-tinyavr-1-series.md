---
path: "/posts/4"
date: "2025-02-27"
title: "Programming tinyAVR 1-Series Microcontrollers on macOS"
icon: "fa-microchip"
redirect_from:
  - /posts/4/
layout: post
---

<img src="/assets/4_0.png" style="width: 100%; height:140px; object-fit: cover;" alt="A microcontroller">

I have found myself having to set up this development environment several times and I keep forgetting how to do it easily. Sharing these steps on how to set up the tooling for these new ICs and the other new series of chips.

First, install `brew` from [brew.sh](https://brew.sh) if you don't have it already, and then install `avr-gcc`:
```
brew tap osx-cross/avr
brew install avr-gcc
```

This installs a recent version of the compiler tools, but lacks libraries and headers for the latest series of microcontrollers. For this, you should clone the official Microchip `avr-libc` repository:

```
git clone https://github.com/avrdudes/avr-libc
```

Now, when you want to compile for a specific IC, you can point your `avr-gcc` to use the relevant files from the repo you just cloned (replace `/path/to/avr-libc` with the actual path you cloned to):

```
avr-gcc \
    -mmcu=attiny412 \
    -I/path/to/avr-libc/include \
    -B/path/to/avr-libc/avr/devices/attiny412 \
    -Wall -Os \
    main.c
```

And finally you can use `avr-objcopy -O ihex a.out main.hex` to generate a proper hex file that is ready for flashing.
