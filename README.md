ModBase is a template project for split disassemblies of old videogame
software. This particular template targets GB and GBC software using the RGBDS
development system.

== Requirements ==

* Latest version of RGBDS
* Python (any version)
* make (probably GNU make)

== Getting started ==

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