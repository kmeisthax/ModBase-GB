ModBase is a template project for split disassemblies of old videogame
software. This particular template targets GB and GBC software using the RGBDS
development system.

## Requirements

* Latest version of RGBDS
* Python (any version)
* make (probably GNU make)

## Getting started

*If you wish to disassemble a game with multiple versions, please take a look
at the section entitled Multiple Base ROMs.*

First, download this project's .zip file and create a new repository from it.
Do not create a GitHub fork for your repository - new build code committed here
is not guaranteed to be relevant or compatible with your project. Do, however,
create a new Git repository to store your code.

Place your base ROM in base/ and alter lines 12-13 of the Makefile to point at
the file. For example, if your file is base/Tumiki_Fighters.gb, then change

    BASEROM := ${BASE_DIR}/baserom.gbc
    BUILDROM := ${BUILD_DIR}/rom.gbc

to

    BASEROM := ${BASE_DIR}/Tumiki_Fighters.gb
    BUILDROM := ${BUILD_DIR}/Tumiki_Fighters.gb

You may now add code to src/ representing your game. As you add code, it will
automatically be included in the project, compiled, and verified against the
base ROM. If you introduce a mistake into your project, you will recieve an
error, such as:

    cmp build/Frozen_Bubble.gbc base/Frozen_Bubble.gbc
    build/Frozen_Bubble.gbc base/Frozen_Bubble.gbc differ: char 14, line 23

You can diagnose these errors using hexdump -C, cmp --verbose, or any graphical
hex editor with a compare feature.

### Multiple Base ROMs

Some games may share code across multiple ROM images. For example, the game may
have a 1.1 revision, or was released in multiple versions, or have multiple
translations. In this case, you will need to modify the build system to be aware
of multiple builds of the ROM image.

First, you will need to declare a BASEROM and BUILDROM for each version of the
game you want to build. Let's say your game had a US and EU release; you'd then
have declarations like so:

    BASEROM_US := ${BASE_DIR}/Tumiki_Fighters_(U).gb
    BASEROM_EU := ${BASE_DIR}/Tumiki_Fighters_(E).gb
    BUILDROM_US := ${BUILD_DIR}/Tumiki_Fighters_(U).gb
    BUILDROM_EU := ${BUILD_DIR}/Tumiki_Fighters_(E).gb

You will then need to duplicate the linking and comparison tasks to reference
the new ROMs. Lines 24-38 of the Makefile contain the rules for linking and
comparing ROMs. The $(BUILDROM) target will need to be duplicated into
$(BUILDROM_US) and $(BUILDROM_EU) targets, with the $(BASEROM) overlay parameter
changed to match the ROM being built. You will also need to change the compare
target to run a comparison between both base ROMs and built ROMs; and change the
rom target to require both ROMs to be built.

Once you have finished these changes, your modifications should resemble the
following example code (comments elided):

    all: rom compare
    
    rom: $(BUILDROM_US) $(BUILDROM_EU)
    
    $(BUILDROM_US): $(OBJS:%.o=${BUILD_DIR}/%.o)
    	rgblink -n $(@:.gbc=.sym) -m $(@:.gbc=.map) -O $(BASEROM_US) -o $@ $^
    	rgbfix -v $@
    
    $(BUILDROM_EU): $(OBJS:%.o=${BUILD_DIR}/%.o)
    	rgblink -n $(@:.gbc=.sym) -m $(@:.gbc=.map) -O $(BASEROM_EU) -o $@ $^
    	rgbfix -v $@
    
    compare: rom
    	cmp $(BUILDROM_US) $(BASEROM_US)
    	cmp $(BUILDROM_EU) $(BASEROM_EU)

This is suitable for building a project where you don't intend to disassemble
any differences between the two ROMs - RGBDS's overlay feature will include them
for you. However, if you do wish to do so, then you need to add different
objects for each file. By convention, we store version-specific files in the
src/versions path, so we configure $(OBJS) to exclude it. We then add EU and US
specific variables, which only pull files from each individual versions'
directories. You will still need to make sure these variables are included in
$(OBJS_RGBASM) so that they will be targeted by the assembler, and to include
each specific version variable as a source for each version's built ROM.

All of that will look like this:

    OBJS := $(patsubst %.asm,%.o,$(shell find src -type f -name "*.asm" -not -path "src/version/**"))
    OBJS_US := $(patsubst %.asm,%.o,$(shell find src/version/us -type f -name "*.asm"))
    OBJS_EU := $(patsubst %.asm,%.o,$(shell find src/version/eu -type f -name "*.asm"))
    OBJS_RGBASM := ${OBJS} ${OBJS_US} ${OBJS_EU}
    
...and further down...
    
    $(BUILDROM_US): $(OBJS:%.o=${BUILD_DIR}/%.o) $(OBJS_US:%.o=${BUILD_DIR}/%.o)
    	rgblink -n $(@:.gbc=.sym) -m $(@:.gbc=.map) -O $(BASEROM_US) -o $@ $^
    	rgbfix -v $@
    
    $(BUILDROM_EU): $(OBJS:%.o=${BUILD_DIR}/%.o) $(OBJS_EU:%.o=${BUILD_DIR}/%.o)
    	rgblink -n $(@:.gbc=.sym) -m $(@:.gbc=.map) -O $(BASEROM_EU) -o $@ $^
    	rgbfix -v $@

With that last change, you are now clear to build two ROMs.

## Adding image resources

Almost every game will have image resources of some kind. ModBase-GB ships with
support for three image resource types:

 * 1 bit per pixel / black-and-white uncompressed (.1bpp)
 * 2 bits per pixel / grayscale uncompressed (.2bpp)
 * 2 bits per pixel / color uncompressed (.color.2bpp / .gbcpal)

All three resource types are built from PNG image files. The usual path for an
image resource is the uncompressed grayscale path, which generates a .2bpp file
that can be directly included in your project. If you have a grayscale PNG at,
say, src/titlescreen/bg.png; then you can include it in an .asm file like so:

    TitleScreen_BGImage::
        INCBIN 'build/src/titlescreen/bg.2bpp'

If your resource is black-and-white, you may instead substitute .1bpp instead
of .2bpp.

### Indexed color prohibition

If you recieve an error such as:

    src/titlescreen/bg.png is an indexed PNG. Please convert it to truecolor or grayscale.

You are tripping up against ModBase's indexed color protection. rgbgfx is known
to generate unexpected results with an indexed color image; so we prohibit it's
use by default.

Strictly speaking, "unexpected results" are caused by the fact that rgbgfx uses
the indexes of the PNG file directly to determine the indexes produced on the
GB side; and *not* the grayscale values of those colors.

### Single-palette color images

If your game contains an image resouce with it's own single palette, then you
can store the image and color data in a single PNG file. You will need to
modify your .asm file to request a color-extracted 2bpp file and palette data:

    Items_ItemImages::
        INCBIN 'build/src/items/gun.color.2bpp'
        ;other images...
    
    Items_ItemPalettes::
        INCBIN 'build/src/items/gun.gbcpal'
        ;other images...

This is useful for, say, a game with many item graphics that contain their own
palette.

The generated palette files will be ordered according to the luminance of each
color, and the generated image data will be indexed to match the palette. If
your game does not fit this pattern, you will need to use an indexed color PNG.
This is the only image format where using indexed color is safe, provided you
ensure that the order of color indexes in the PNG match the indexes used by the
game. You will need to make sure that your indexed color PNG contains exactly
four entries in it's palette, and no more.