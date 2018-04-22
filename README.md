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