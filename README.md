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

### Convention is important!

By our default Makefile configuration, src/ holds all disassembled game code.
Furthermore, standard convention is to store all code and assets within specific
component folders. For example, if your game had a titlescreen with image assets
and UI code, then you would have the following files:

    src/titlescreen/state_machine.asm
    src/titlescreen/resources.asm
    src/titlescreen/background_gfx.png
    src/titlescreen/sprite_gfx.png
    
The purpose of grouping code and coupled assets together in component
directories is to clearly indicate the relationship between the two. For the
same reason, any labels declared in *.asm files should be prefixed with the
name of the component directory the file exists within. This is so that
references from other *.asm files to this symbol will always indicate what
directory a symbol's code is. Furthermore, exported labels should be CamelCased
with a single _ separating the component from the rest of the label; while local
labels should be always lowercase with an _ in lieu of spaces between words.
Don't be afraid of long labels.

Avoid long files with large amounts of code. The longer the file, the harder for
people to scan through it. At the same time, a single *.asm file must have a
clearly indicated purpose. If your game has a state machine, then you create a
state_machine.asm file and put the state table and code in there. If your game
has a custom scripting language with 300 opcodes, you may want to split up their
implementations according to purpose. Perhaps something like:

    src/scriptvm/opcodes/arithmetic.asm
    src/scriptvm/opcodes/resource_ldr.asm
    src/scriptvm/opcodes/sprite_choreo.asm
    src/scriptvm/opcodes/playfield.asm

This also demonstrates how you can group files within a subdirectory of a
component. Just as long files are a detriment to readability, so are long
directories. You do not have to prefix labels with both directories, however.

Code within an *.asm file must be formatted correctly. Labels should be always
indicated; there should be no memory locations scattered throughout the code.
All code should be indented with four spaces. An empty line should be added
after every string of instructions that write memory or alter control flow.
Conditional branches should include a label for the branch not taken. Don't be
afraid to rename labels or do other sweeping refactors if the current set of
labels don't accurately describe the function of the code. Use comments, but
only to explain the purpose of an exported label, or where the behavior of the
code isn't obvious from the labels and instructions in use.

What looks more readable to you?

    .start ;this runs every frame
    call $2B2B
    call statemachine
    ld a, [$C420]
    jr z, .nothing_todo
    call linktx
    .nothing_todo
    halt ;this stops the game until an interrupt happens
    jr .start
    
Or?

    .game_loop
        call LCDC_ExecDMA
        call Game_StateMachine
        
        ld a, [W_SIO_Connected]
        jr z, .no_link_connection
        
    .link_connection
        call SIO_RunLinkTxDriver
        
    .no_link_connection
        halt
        jr .game_loop

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