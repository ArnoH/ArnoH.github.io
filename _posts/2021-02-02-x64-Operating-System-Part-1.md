---
layout: single
title:  "x64 Operating System Part 1: Environment Setup and Stage 1 Bootloader"
date:   2020-02-02 13:57:40 +0200
categories: ArnOS
tags: ArnOS OSDev bootloader
---
Ever since I was a little kid I have been facinated by the inner workings of everything. This eventually lead to me persuing an Engineering degree, but what facinated me the most of all since early on were computers and everything related to them. Initially this interest was more with specific software, but as I learned more I found out that most of the heavy lifting was always done by the layers below them.

Since learning that I gave myself the goal of building up the layers myself from scratch. This means not only writing basic versions of the cool software and libraries I have been interested in, but also creating a basic OS they will be built upon to prove to myself that I can do it. Technically the operating system is not the lowest level I could go, but for the software side it should be enough. There is a good chance I will go further in the future and go a layer deeper: designing a custom CPU, but that is for another day.

## The Plan
I like to challenge myself for each project that I do. This combined with the fact that I do this project to learn about every piece of the internals of the project means that I will try not to use any external libraries. There may be some exceptions to this for quality-of-life improvements, but this will be kept to a minimum.

As an extra added challenge I will be using this project to learn [Rust](https://www.rust-lang.org/). This means this series should not be seen as a guide to write a quality OS in Rust, but as an educative journey in which I will make mistakes (and hopefully learn from them) so you don't have to make the same ones. It can still be useful as a guide as long as everything I say is taken with a grain of salt. With the previously mentioned restriction of not using any external libraries, this means that only the Rust core library and the compiler-buitlins crate will be used as a representation of "standard Rust".

The plan for this OS is to be a 64bit OS for the [`x86` architecture](https://en.wikipedia.org/wiki/X86). [POSIX](https://en.wikipedia.org/wiki/POSIX) compliance is also a goal to make it easier to port existing applications, but this could be achieved using a compatibility layer instead of designing the whole OS around it. Since this is an educational project and not a serious attempt at creating the next big OS, not a lot of planning will be done beforehand for the full structure of the OS. I will mostly design features when it is time to implement them and will only have a vague idea of them before that.

Another choice I made is to write my own [BIOS](https://wiki.osdev.org/BIOS) bootloader instead of using an existing one or using [UEFI](https://wiki.osdev.org/UEFI). While using BIOS is considered outdated, I think it still has a lot of educational value. Adding support for other boot systems might happen in the future.

Since I use Arch Linux as my main development OS, that is also what I will use here. Most of the things should be the same for other platforms except for the setup and external tools used.

Since I am bad with comming up with names for my projects, I usually use some kind of variation on my own name. It is always easy to rebrand later if needed, but it saves me the time of thinking of a good project name while always having to worry that some other project already exist with the cool name I came up with (as is often the case). In this case I decided on starting with the name `ArnOS`. There don't seem to be easily findable projects with that name, so I just have to live with people misreading the name and thinking it has something to do with `ARM`.

## Acknowledgements
Since this is an educative project for myself, I will of course be using a lot of different sources for inspiration and information. I will try to acknowledge the sources I use for every post. In general a lot of the information on operating systems will come from the [OSDev Wiki](https://wiki.osdev.org/Main_Page).

Aditionally some Rust specific OS development resources will also be used. My most important resource for this is Philipp Oppermann's blog [Writing an OS in Rust](https://os.phil-opp.com/).

For the bootloader code itself, the source code of some existing bootloaders was consulted. This was mostly to check for possible edge cases I forgot to handle or to find possible solutions to problems I ran into. The bootloader I mostly used as a reference due to its simplicity is the [BOOTBOOT x86_64-bios Reference Implementation](https://gitlab.com/bztsrc/bootboot/-/tree/master).

## Basic Setup
### Tools
The build scripts of the project will be in the form of `Makefiles`, so [GNU Make](https://www.gnu.org/software/make/) will be used. The bootloader will be written in `x86 assembly` and compiled using [NASM](https://www.nasm.us/). Finaly, the boot image will be made using `genisoimage` available from the `cdrtools` package on Arch. I don't see this as an external library breaking my self imposed rule since this is more like a *compiler* for the boot image. Testing the OS will be done using [QEMU](https://www.qemu.org/).

### Basic Makefiles
When starting a project, I want to be able to test and iterate on initial designs as quickly as possible. Because of this I usually spend a lot of time initially on setting up my environment instead of starting with writing code I won't be able to test yet. For that reason we will start setting up most of the environment before writing any bootloader code.

First of all we make a `scripts` directory to contain the scripts we will use common to the different modules of the project. In this directory we will make a `config.mk` file, which will for now be the only thing in this directory. Later it might be a good idea to extract things like making the bootable image to scripts, but for now that is not necessary.

{% highlight makefile %}
# scripts/config.mk
MOD_BOOTLOADER  := bootloader
MOD_KERNEL      := kernel

DIR_BUILD       := $(DIR_ROOT)/build
DIR_DEPS        := $(DIR_ROOT)/deps
DIR_SCRIPTS     := $(DIR_ROOT)/scripts

DIR_ISO         := $(DIR_BUILD)/iso
DIR_ISO_BOOT    := $(DIR_ISO)/boot

OS_NAME         := ArnOS

QEMU        := qemu-system-x86_64
AS          := nasm
AS_FLAGS    := -f bin
MAKE_FLAGS  := --silent
{% endhighlight %}

In the first two lines, the names of the different modules are defined. For now these are only the `bootloader` and the `kernel`, but later there will be other modules like user space tools and libraries.

Afterwards, three special directories are defined: the `build` directory which will contain all the compiled files and executables, the `deps` directory which will contain all the automatically generated dependency information for the assembly files, and the `scripts` directory which this file is inside of. These directories are relative to a directory speciefied by the `$(DIR_ROOT)` variable which is not included here. The files including this config should define this variable before including this config.

`DIR_ISO` is the directory which will be the base directory of the [iso](https://en.wikipedia.org/wiki/ISO_9660) boot image generated for the OS. The files and directory structure of this directory will be copied into the iso file.

The name of the operating system is set in the `OS_NAME` variable. This variable will be used to generate filenames. For example the filename of the boot image will be based on this variable.

The used assembler (`nasm`) is set as the assembler command `AS`.

The assembler flag specified in the `AS_FLAGS` variable (`-f bin`) denotes that raw binary files should be generated with the assembled instruction in them. These files will thus only contain executable code and data without being embedded in another file format.

The flags that will be passed to the `make` commands of the different modules are specified in the `MAKE_FLAGS` variable. Here the passed `--silent` flag suppresses the echoing of commands while executing them while also preventing the `Entering/Leaving directory...` messages to decrease the verbosity of the output.

In our root directory we can create the root makefile called `Makefile`. This will be the biggest makefile that we will make today.

{% highlight makefile %}
# Makefile
DIR_ROOT := .

include $(DIR_ROOT)/scripts/config.mk

ISO_FILE_CD := $(DIR_BUILD)/$(OS_NAME)-cd.iso

default: iso_cd
all: iso_cd

.PHONY: default all bootloader kernel iso clean cleandeps cleanall debug debugwait

bootloader:
    @$(MAKE) $(MAKE_FLAGS) --directory=$(DIR_ROOT)/$(MOD_BOOTLOADER)
kernel:
    @$(MAKE) $(MAKE_FLAGS) --directory=$(DIR_ROOT)/$(MOD_KERNEL)

iso_cd: $(ISO_FILE_CD)
    @echo "[iso] ISO file ready at '$(ISO_FILE_CD)'"
    
$(ISO_FILE_CD): bootloader kernel
# This will be filled in later

clean:
    @rm -rf $(DIR_BUILD)
    @echo "Build files deleted"

cleandeps:
    @rm -rf $(DIR_DEPS)
    @echo "Dependency files deleted"

debug: iso_cd
    @$(QEMU) -gdb tcp::2345 -cdrom $(ISO_FILE_CD)

debugwait: iso_cd
    @$(QEMU) -S -gdb tcp::2345 -cdrom $(ISO_FILE_CD)

cleanall: clean cleandeps
{% endhighlight %}

First of all, we define the `DIR_ROOT` variable as the current directory `.`. Following that we include the `config.mk` file that we created earlier. This will allow us to use the variables defined in there.

`ISO_FILE_CD` is a variable denoting the path to the boot image that should be generated. Here it evaluate to `./build/ArnOS-cd.iso`.

The line `default: iso_cd` denotes that the `iso_cd` target should be used as the default target which is used when no target is given to the `make` command. The `all: iso_cd` line analogously denotes that the `iso_cd` target should be used when `make all` is executed.

The `.PHONY` rule is used to denote that all targets in the rule are not actual files. Otherwise the existence of a file called `bootloader` in the root directory for example could prevent the `bootloader` rule from being executed.

The `bootloader` and `kernel` targets just redirect to their respective makefiles.

The `iso_cd` target just runs the target corresponding to the bootable ISO image file while echoing a helpful message stating the ISO file exists and the path to the file. For the ISO image, both the kernel and the bootloader are dependencies since they should be included in the boot image. The details on the generation of the image will be filled in after the bootloader is configured.

`clean` and `cleandeps` just delete the `bin` and `deps` directories respectively while `cleanall` is a shorthand for doing both.

`debug` and `debugwait` both start up QEMU with a gdb server started on `localhost:2345`. `debugwait` also halts execution so there is time to attach the debugger before the start of the system. This is useful for debugging the bootloader for example.

### Bootloader Setup
Everything related to the bootloader will live inside a directory called `bootloader`. The bootloader will be written entirely in `x86 assembly`. As such, the only thing the makefile should do is assemble the `*.asm` files, and take care of eventual dependecy chains to know when files should be reassembled.

{% highlight makefile %}
DIR_ROOT	:= ..

include $(DIR_ROOT)/scripts/config.mk

DIR_BUILD   := $(DIR_BUILD)/$(MOD_BOOTLOADER)
DIR_DEPS    := $(DIR_DEPS)/$(MOD_BOOTLOADER)

ASM_FILES   := $(wildcard *.asm)
BIN_FILES   := $(ASM_FILES:%.asm=$(DIR_BUILD)/%.bin)
DEP_FILES   := $(ASM_FILES:%.asm=$(DIR_DEPS)/%.dep)

TAG := "[$(MOD_BOOTLOADER)]"

all: $(BIN_FILES)
    @echo "$(TAG) Successfully built"

.PHONY: all make_dirs clean

make_dirs:
    @mkdir -p $(DIR_BUILD) $(DIR_DEPS)

$(BIN_FILES): $(DIR_BUILD)/%.bin:%.asm | make_dirs
    @echo "$(TAG) Assembling $<"
    @$(AS) $(AS_FLAGS) $< -o $@

$(DEP_FILES): $(DIR_DEPS)/%.dep:%.asm | make_dirs
    @echo "$(TAG) Generating dependencies for $<"
    @rm -f $@
    @$(AS) $(AS_FLAGS) -M -MT @@@ $< > $@.tmp
    @sed 's,@@@[ :]*,$(DIR_BUILD)/$*.bin $@: ,g' < $@.tmp> $@
    @rm -f $@.tmp

clean:
    @rm -rf $(DIR_BUILD)
    @rm -rf $(DIR_DEPS)

-include $(DEP_FILES)
{% endhighlight %}

As usual, the file starts with setting the `$(DIR_ROOT)` and including the config. Afterwards we overwrite `$(DIR_BUILD)` and `$(DIR_DEPS)` to point to a subdirectory inside of them named after the current module (`bootloader`).

Next the list of source files is created. `$(wildcard *.asm)` evaluates to the list of files in the current directory with the extension `.asm`.

The resulting binary filenames and dependecy filename are generated using substitution references. `$(ASM_FILES:%.asm=$(DIR_BUILD)/%.bin)` evaluates to a list where every value of the `$(ASM_FILES)` list was processed based on the substitution rule `%.asm=$(DIR_BUILD)/%.bin`. This rule would for example turn `test.asm` into `$(DIR_BUILD)/test.bin`.

The `$(TAG)` variable is used to prepend the printed text of this module. For this module this results in `[bootloader]`.

For this module, building everything means building evey binary file that has a corresponding source file. This is why the `all` target requires the list of binary files (`$(BIN_FILES)`) as prerequisite. This time no `default` target is specified, but in that case the first target of the file not starting with a dot will be chosen as the default. In this case that will be the `all` target.

As usual, all named targets are declared as phony.

the `make_dirs` target creates the build and dependency directories based on the variables defined earlier. The `-p` argument of `mkdir` denotes that non-existing parent directories should also be created and that no error should be given when the directory already exists.

For the binary files the rule is defined as follows:
{% highlight makefile %}
$(BIN_FILES): $(DIR_BUILD)/%.bin:%.asm | make_dirs
{% endhighlight %}

This rule will be split up for each entry in the `$(BIN_FILES)` list. The first prerequisiste is the source file you get by applying the substitution rule `$(DIR_BUILD)/%.bin:%.asm`. This results in the opposite effect of the substitution rule mentioned earlier in the definition of `$(BIN_FILES)`: it turns `$(DIR_BUILD)/test.bin` back into `test.asm`.

The pipe (`|`) denotes that the following prerequisites are `order-only prerequisistes`. These prerequisites (in this case only `make_dirs`) won't cause the target to be rebuilt if they have to be run. Since `make_dirs` is not a file, it will always run and we don't want to reassemble the binary just because of this, but only when a dependency changes. We still want it to be a prerequisite, but we don't want to rebuild because of it and that is what an `order-only prerequisite` is.

Inside a `recipe` of a rule, there are a couple of special variables that can be used. The most important ones for us are `$@` evaluating to the target (in this case the `.bin` file), and `$<` evaluating to the first prerequisite (in this case the `.asm` file). For the binary files we print a message saying which file we are assembling and then we assemble it. The `-o $@` parameter to `nasm` denotes the output file of the command.

For the dependency files a similar approach is used as for the binary files. Instead of assembling the files, the dependencies are extracted from the source file. This is done based on [part of the manual for make](https://www.gnu.org/software/make/manual/html_node/Automatic-Prerequisites.html). This recipe works in a couple of steps:

* `rm -f $@` deletes the dependency file if it already exists.
* The next command gets the dependencies from `nasm` by passing in the `-M` argument. The `-MT @@@` argument overwrites the target name for the dependency with `@@@` to replace later.
* The `sed` line replaces the `@@@` with the binary file path and the path to the dependency file itself. This last part is why this temporary substitution of `@@@` was needed. Adding this means that when this dependency file is included in the makefile, the dependencies of the binary file will also become dependencies for the dependency file itself so it will know to regenerate when one of those changes.
* This substitution is done through a temporary file because of the way the redirections are done with the `sed` command. This file is the target with `.tmp` appended to it.

Cleaning is done by deleting the build and dependency directories.

Finally, the dependency files are included in this makefile. The dash in front of the include directive means that no errors should be reported if they occur (if one of the files doesn't exist for example). This is the wanted behavior since if an error occurs with making one of the dependency files, it will already be reported during generation. It is also this include line which will build the dependency files as a target. The dependency files themselves are layed out as make rules. As an example, this is the dependency file that will be generated for one of the binary files of the bootloader:

{% highlight makefile %}
# deps/bootloader/cdboot.dep
../build/bootloader/cdboot.bin ../deps/bootloader/cdboot.dep: cdboot.asm include/common.inc
{% endhighlight %}

This file has two targets, `cdboot.bin` and `cdboot.dep`, the last of which is also the dependency file itself. This rule will as such be split up into two different rules, one for each target. The dependencies of the rule are the source file `cdboot.asm` as well as a file to be included in the source file `include/common.inc`. These will be created later.

## Stage 1 Bootloader
Since the kernel will be written in rust, a lot more configuration is needed for that, but the configuration that we have done so far is already enough to create most of the bootloader. That is why I opted to start on the bootloader before doing the setup for the kernel.

### Bootloader structure
When the BIOS tries to boot a bootable medium (like our ISO image), it will start by only loading the first sector, the [Boot sector](https://en.wikipedia.org/wiki/Boot_sector), into RAM. This sector is usually 512 bytes in size for conventional drives, but for our ISO image this will be 2048 bytes. Since we will eventually also support other types of boot drives, we will try to make the code of the first sector fit into the smaller 512-byte size to be able to reuse more of the code.

Because only one sector is loaded, and the fact that it would be hard for the whole bootloader to fit into 512 bytes, the bootloader is split up into multiple stages. The stage corresponding to the first sector is the `Stage 1 Bootloader`. Its only responsibility is loading the `Stage 2 Bootloader` from the drive to RAM. This `Stage 2 Bootloader` is then responsible for going from `Real Mode` to `Long Mode` (explained in the next section), loading the `kernel` from the drive, decrypting it if needed, and jumping to the entry point of the kernel. These could be divided up into extra stages, but for this project the complexity of these substeps will be so low that it doesn't make sense to call these different stages.

### Operational Modes
When the CPU starts execution, it will do so in [Real Mode](https://wiki.osdev.org/Real_Mode). This is done for backwards compatibility reasons. The biggest limitations of this operational mode are that only 16-bit addressing is possible, and that of that small amount of addressable memory a lot is used up by other things (as can be seen [here](https://wiki.osdev.org/Memory_Map_(x86))). The positive thing about `Real Mode` is that the `BIOS` interrupts can be used which implement basic drivers for things like the filesystem. This is useful to load things from a drive into RAM.

The next operational mode is [Protected Mode](https://en.wikipedia.org/wiki/Protected_mode). This is a 32-bit mode allowing 32-bit addressing as well as some other features like [Paging](https://en.wikipedia.org/wiki/Paging). We will mostly use this mode as a stepping stone to [Long Mode](https://wiki.osdev.org/Long_Mode), but we will also load the kernel into memory in this mode. This is because to do this easily, a lot of switching back to `Real Mode` is needed which is easier to do from `Protected Mode`. The reason we can't do this in `Real Mode` is because the kernel will be too large in general to fit in the limited address space of `Real Mode`. To overcome this, we will load sections in `Real Mode` and relocate these to their correct place in `Protected Mode`.

### Stage 1 Memory Map
As mentioned in the previous section, we start in `Real Mode` with [limited addressable and usable memory](https://wiki.osdev.org/Memory_Map_(x86)). The region that is (and will stay) available is the region between `0x500` (inclusive) and `0x80000` (exclusive). This is the only memory we can safely edit in any way. The only things we need space for are: the stack, the `Stage 1 Bootloader` and the `Stage 2 Bootloader`.

The stack doesn't need to be large since it won't be used much except for the occasional call to a subroutine or for some register manipulation. Because of this, a 256 byte region is reserved for this starting from `0x500`.

The `Stage 1 Bootloader` will be limited to 512 bytes as mentioned before. As a result it will occupy the region `0x600-0x7FF` (both inclusive). Initially, the BIOS will load this at address `0x7C00`, so we will have to relocate it to our desired location.

The `Stage 2 Bootloader` is theoretically only limited to the amount of bytes that are left. There are also practical requirements to consider, namely the maximum amount of bytes that can be read at a time. The lowest limit I know of is 127 sectors for [some Phoenix BIOSes](https://en.wikipedia.org/wiki/INT_13H#INT_13h_AH=42h:_Extended_Read_Sectors_From_Drive). In the worst case, these will be 512-byte sectors. Since we will work with 2048 byte sectors, we will limit ourselves to a size of 31 CD sectors of 2048 bytes (equivalent to 124 sectors of 512 bytes). This means the `Stage 2 Bootloader` will occupy the region `0x800-0xFFFF` (both inclusive).

To summarize, here is an overview of the memory map we will use:

| Region (incl.) | Contents |
| ----------- | ----------- |
| 0x0500-0x05FF | Stack |
| 0x0600-0x07FF | Stage 1 Bootloader |
| 0x0800-0xFFFF | Stage 2 Bootloader |

###  Loading the Stage 2 Bootloader
In order to more easily load the `Stage 2 Bootloader`, a feature of the [`El Torito`](https://wiki.osdev.org/El-Torito) extension to the [`ISO 9660`](https://wiki.osdev.org/ISO_9660) filesystem is used: the possibility to embed a `Boot Information Table` in the boot sector. This table will contain things like the sector number of the boot file in the filesystem. We can embed the `Stage 2 Bootloader` in the same file as the boot file (the `Stage 1 Bootloader`) at a constant offset. This allows for us to know the sector at which the `Stage 2 Bootloader` starts which makes it easy to load.

### Writing the Stage 1 Bootloader
#### Relocating the Boot Sector
We will create a file called `cdboot.asm` in the `bootloader` directory. This file will contain the `Stage 1 Bootloader`.

{% highlight nasm %}
; bootloader/cdboot.asm

; The first-stage boot loader starts in 16-bit real mode.
bits 16

; We will relocate to 0x0600
; so the code has to be relative to that address
org 0x0600

boot:
    jmp short init

    db  8-($-$$) dup 0
bi_pvd:         dd 0
bi_file:        dd 0
bi_length:      dd 0
bi_csum:        dd 0
bi_reserved:    db 50 dup 0

init:
{% endhighlight %}

The first instruction that will be executed is a short jump to the `init` label defined at the bottom. This is because the `Boot Information Table` will be embedded into the binary starting from byte 8. A short jump is specified because it is important that every jump is relative instead of absolute since we haven't relocated ourselves to the address we think we are.

Space for the `Boot Information Table` is allocated starting from byte 8 using the `db 8-($-$$) dup 0` construct. This means that 8 bytes with value 0 have to be inserted there minus the difference between the current address and the start address of the file. Here this means 8 zero bytes will be inserted minus the amount of bytes already inserted by the `jmp short` instruction. 

Of these fields, only `bi_file` and `bi_length` will be used. `bi_file` is the sector number that this file starts at in the filesystem, given as an [LBA](https://en.wikipedia.org/wiki/Logical_block_addressing) address. One sector further will be the start of the `Stage 2 Bootloader`. `bi_file` is the length of the file in bytes.

We can then continue to write the rest of the self relocation code.

{% highlight nasm %}
; bootloader/cdboot.asm

init:
    ; Disable interrupts since we modify the the stack pointer
    cli
    
    ; Clear Direction Flag
    cld

    ; Point stack to the desired address
    xor ax, ax
    mov ss, ax
    mov sp, 0x600
    
    ; It should be safe to re-enable interrupts now
    sti

    ; Set es to 0 so di points to a true address
    mov es, ax

    ; Set ds to cs so si can point to our code
    push cs
    pop ds

    ; Get address of our code within the code segment
    call .getAddress

.getAddress:
    pop si
    sub si, .getAddress-boot

    ; Copy self to new relocated address
    mov di, 0x600
    mov cx, 0x80
    rep movsd

    ; Set ds to 0 so we can reference variables again in si
    mov ds, ax

    ; Do a far jump to jump to the desired absolute address
    jmp 0:.start

.start:
{% endhighlight %}

As initial setup, interrupts are disabled since we will change the location of the stack. Afterwards we re-enable it. The `Direction Flag` is cleared so when using repeated [String Operations](https://www.felixcloutier.com/x86/rep:repe:repz:repne:repnz), the source and destination indices will increase instead of decrease. The use of these instructions is also the reason the `Extra Segment` (`es`) is set to zero. String operations act with `ds:si` as a source and `es:di` as a destination. Likewise, the `Data Segment` (`ds`) is set to the `Code Segment` (`cs`) since we will copy the code from the code segment to the data segment.

In `Real Mode`, segments denote a constant offset that is added to the register values when addressing things depending on the context. You can read more about this [here](https://wiki.osdev.org/Segmentation).

We get the base address of the code we are running by doing a call to a label. This is a `near call` meaning it is done based on an offset to the current instruction pointer instead of an absolute address. This call instruction pushes the return address to the stack which we pop in the `si` register. This should correspond to the absolute address of the `.getAddress` label within the code segment since that is where the instruction after the call instruction is.

As a result, to get the base address of the code we have to subtract the offset of this label relative to the start of the code (the `boot` label).

Then we can use a `movs` instruction with a `rep` prefix to repeat it N times based on the value in `cx`. The suffix `d` means that we move with a `dword` at a time, meaning 4 bytes. That is the reason why we put `0x80` into `cx` as this is 128 (512 divided by 4).

We then again clear the data segment so `si` can point to absolute offsets again, and we can do a far jump to jump to an absolute address. Starting from this point we are located at our desired address and can start actually loading the second stage.

#### Loading the Second Stage
The BIOS already comes with the needed device drivers to load data from our CD to RAM. These features are available through the use of `int 13h`. When calling this interrupt, certain parameters can be passed through the use of certain registers. The most important parameter is the requested function, passed through the `ah` register. This parameter tells the BIOS what operation we want to do. I will write specific functions of `int 13h` with the following notation: `int 13h; ah=XXh` with `XX` being the value of `ah` passed in hexadecimal.

The basic `int 13h` functions work based on [`Cylinder-Head-Sector` or `CHS`](https://en.wikipedia.org/wiki/Cylinder-head-sector) addressing. Working with this addressing scheme is a bit more work than using the alternative `Logical Block` addressing as the latter are just linear sequential addresses. This addressing mode was implemented in the form of an extension of `int 13h`, but as this already dates from the mid 90' it shouldn't be a problem to only support devices implementing this extension.

The way to check if this extension is implemented is by using [`int 13h; ah=41h`](https://en.wikipedia.org/wiki/INT_13H#INT_13h_AH=41h:_Check_Extensions_Present). After calling this function, we check if `Device Access using the packet structure` is supported.

{% highlight nasm %}
; bootloader/cdboot.asm

; <Old code up here>
.start:
    mov [drive], dl

    mov ah, 0x41
    mov bx, 0x55AA
    clc
    int 0x13

    jc .failLBA
    cmp bx, 0xAA55
    jne .failLBA
    test cl, 1
    jnz .lbaSupported

.failLBA:
    mov si, strTooOld
    jmp abort

.lbaSupported:
    ; <Code will be added>

abort:
    ; <Code will be added>

; Variables and Constants
drive:      db 0

strTooOld:  db  "System is too old to boot on!", 0
{% endhighlight %}

Before doing anything else, we will save the boot drive index into a variable defined at the bottom. The definition is done using `db 0` which places a byte with the value `0` at that location. With the `drive:` label pointing to that location, it is effectively a variable.

When the BIOS passes control to us, it puts the boot drive index into the `dl` register. This register wasn't used yet in the code, so it still contains this value. That is also why we don't need to set a value in it for `int 13h; ah=41h`. We save this value into a variable because when calling BIOS interrupts there is always a chance that some BIOSes will overwrite register values.

Since we now have a failure condition, we will need an `abort` function which will print a message to the screen, wait for user input (so they have time to see the message), and reset the BIOS. This code will look as follows:

{% highlight nasm %}
; bootloader/cdboot.asm

; <Old code up here>

; Abort function prints string pointed to by si and shuts down
abort:
	call print

	; Wait for user input (so they can read the message)
	xor ax, ax
	int 0x16

	; Jump to Bios Reset vector
	jmp 0xFFFF:0

print:
	mov ah, 0x0E
	mov bx, 11
.loop:
	lodsb
	test al, al
	jz .end
	int 0x10
	jmp .loop
.end:
	ret

; Variables and Constants
drive:      db 0

strTooOld:  db  "System is too old to boot on!", 0
{% endhighlight %}

Waiting on user input is done through the use of `int 16h; ah=00h` which reads a key press. If a key was already pressed before this interrupt is used, that key will immediately be returned, but that shouldn't be a problem here due to the small time frame between startup and the error message.

Resetting is done by jumping to the `Reset Vector` of the BIOS. This is the same adress that is executed when the processor starts up. This will effectively be the same as a restart.

Now we can prepare loading the second stage. Interrupt [`int 13h; ah=42h`](https://en.wikipedia.org/wiki/INT_13H#INT_13h_AH=42h:_Extended_Read_Sectors_From_Drive) will be used. This interrupt will read sectors based on the data in a structure we pass it. We will define a structure, instantiate it as a variable and then pass a pointer to it in `ds:si`. Since `ds` is set to `0`, `si` will contain the absolute address of the structure instance in memory.

The `Disk Address Packet` structure is layed out as follows:

| Field | Size |
|---|---|
|size of the structure (16)| 1 byte|
|reserved (0)| 1 byte|
|number of sectors to read| 2 bytes|
|`segment:offset`| 4 bytes|
| start sector index | 8 bytes|

Since the size of the structure is a fixed `16` bytes, we will merge the first two fields into a 2 byte field and keep the name `size`. Defining the struct and instantiating it as a variable is done using this syntax:

{% highlight nasm %}
; bootloader/cdboot.asm

; The first-stage boot loader starts in 16-bit real mode.
bits 16

; We will relocate to 0x0600
org 0x0600

; Disk Address Packet (DAP) struct (for int 13h; ah=42h)
struc DAP

.size:      resw 1
.count:     resw 1
.address:   resd 1
.sector:    resq 1

endstruc

boot:
; <Old code down here>

; <Rest of the file up here>
strTooOld:  db  "System is too old to boot on!", 0

align 8
diskAddressPacket:
istruc DAP

at DAP.size,    dw 0
at DAP.count,   dw 0
at DAP.address, dd 0
at DAP.sector,  dq 0

iend

{% endhighlight %}

The `struc` and `endstruc` keywords mark the beginning and ending of the structure definition (which we called `DAP` here).

The `istruc` and `iend` keywords mark the beginning and ending of the structure instantiation. The `at DAP.XXX` denote that what comes after (the `dX 0`) is done at the offset of field `XXX` in the `DAP` structure. The `0` is just the initial value in memory. We call the variable `diskAddressPacket`.

We align the structure instance on an 8 byte boundary because it is best to restrict memory accesses to addresses aligned with their operand size. Reading a 32-bit value should only be done on addresses a multiple of 4 for example. Since there is an 8 byte field in here, we align it on an 8 byte boundary. Since we are in `Real Mode`, it is impossible to do an 8 byte read, but it is still a good practice to follow.

With this structure in place, we can fill it in now.

{% highlight nasm %}
; bootloader/cdboot.asm

; <Old code up here>

.lbaSupported:
    mov byte [diskAddressPacket+DAP.size], 0x10
    mov byte [diskAddressPacket+DAP.address+1], 0x08
    mov ebx, dword [bi_file]
    inc ebx ; Stage 2 starts from the second sector of the file
    mov dword [diskAddressPacket+DAP.sector], ebx

    mov eax, dword [bi_length]

    ; Force rounding up to multiple of sector size
    add eax, 2047
    ; Divide by sector size
    shr eax, 11
    ; Now ax contains the number of sectors to in the file

    ; Subtract the first sector because that is this file
    dec ax
    cmp ax, 31 ; 31*2048=124*512
    jbe .sizeOk

    mov si, strTooLarge
    jmp abort

.sizeOk:
    mov word [diskAddressPacket+DAP.count], ax
	
    ; <More code will be added>
    
    ; <Mode code down here>
    
strTooOld:      db  "System is too old to boot on!", 0
strTooLarge:    db  "Stage 2 bootloader is too large!", 0
    ; <Rest of the file down here>
{% endhighlight %}

Note that although we are in 16-bit `Real Mode`, we are still able to use 32-bit instructions.

Things to note are that we start reading one sector later than what `bi_file` contains because the first sector of the file will contain this first stage which is already loaded. For the same reason, we subtract one sector from the size of the file in `bi_length` after its conversion to a number of sectors (rounded up). Finally we check if the second stage is not too large. As mentioned earlier, we limit ourselves to 124 `512-byte sectors` equivalent to 31 `2048-byte CD sectors`.

Now the only thing left is reading the sectors, veryfying that the file was loaded to the correct location and jumping to the entry point.

{% highlight nasm %}
; bootloader/cdboot.asm

; <Old code up here>

.sizeOk:
    mov word [diskAddressPacket+DAP.count], ax

    ; Read the sectors
    ; INT 13h AH=42h: Extended Read Sectors From Drive
    ; dl = drive index
    ; ds:si is pointer to Disk Address Packet
    mov ah, 0x42
    mov si, diskAddressPacket
    mov dl, [drive]
    int 0x13

    ; Check if correctly written
    mov eax, dword [0x800]
    cmp eax, MAGIC
    je .copyOk
    mov si, strCopyFailed
    jmp abort

.copyOk:
	mov ax, word [0x804] ; Entry point of Stage 2
	jmp ax

; <More code down here>

strTooOld:      db  "System is too old to boot on!", 0
strTooLarge:    db  "Stage 2 bootloader is too large!", 0
strCopyFailed:  db  "Error copying stage 2 bootloader!", 0

; <Rest of the file down here>

{% endhighlight %}

Note that `MAGIC` was not defined here. We will define it in a file that both stages will include.

After comparing this magic number read from the first 4 bytes of the file, we jump to the address read from the 2 bytes after the magic number.

{% highlight nasm %}
; bootloader/include/common.inc

%define MAGIC 0xC0DEDAB5

{% endhighlight %}

`MAGIC` is an arbitrary constant that will be placed in the beginning of the second stage to signal that the file was loaded in the correct place. This doesn't verify that the whole file was loaded, but that is good enough. We will include it by adding an include directive to `cdboot.asm`:

{% highlight nasm %}
; bootloader/cdboot.asm
; The first-stage boot loader starts in 16-bit real mode.
bits 16

; We will relocate to 0x0600
org 0x0600

%include "include/common.inc"

; <Rest of the file down here>

{% endhighlight %}

## Building the ISO Image
### Creating a Stage 2 Stub
In order to test if our `Stage 1 Bootloader` works, we need a stage 2 stub to load in. We will just print a message to the screen showing that we successfully loaded our second stage and managed to enter it

{% highlight nasm %}
; bootloader/loader.asm

org 0x800

%include "include/common.inc"

loader_start:
magic:			dd MAGIC
entryPoint:		dw _realEntryPoint

_realEntryPoint:
    mov si, strHello
    jmp abort
    
    ; Abort function prints string pointed to by si and shuts down
abort:
    call realPrint

    ; Wait for user input (so they can read the message)
    xor ax, ax
    int 0x16

    ; Jump to Bios Reset vector
    jmp 0xFFFF:0

realPrint:
    mov ah, 0x0E
    mov bx, 11
.loop:
    lodsb
    test al, al
    jz .end
    int 0x10
    jmp .loop
.end:
    ret
    
strHello: db "Hello From Stage 2!", 0

{% endhighlight %}

The `abort` and `realPrint` functions are based on their `Stage 1` equivalents.

The important thing is that the first thing in the binary is set to be the `MAGIC` number we include from `inlude/common.inc`. After those 4 bytes, we put the (absolute) address of the entry point. This is based on that this file will be put at `0x800` as set by the `org` directive.

### ISO Image Generation
Now that all files are in place and we have decided what the boot file layout should look like, we can actually generate the image and test it.

{% highlight makefile %}
# Makefile

# <Old content up here>

$(ISO_FILE_CD): bootloader kernel
# Note: $(DIR_BUILD) will already exist at this point
	
# Check if file sizes are valid
    test `wc -c <$(DIR_BUILD)/$(MOD_BOOTLOADER)/cdboot.bin` -le 512;
# 63488 = 31*2048
    test `wc -c <$(DIR_BUILD)/$(MOD_BOOTLOADER)/loader.bin` -le 63488;
	
# Remove build/iso if it already exists
    @rm -rf $(DIR_ISO)
    @mkdir -p $(DIR_ISO_BOOT)
	
    @cp $(DIR_BUILD)/$(MOD_BOOTLOADER)/cdboot.bin $(DIR_ISO_BOOT)/boot.bin
	
# Resize cdboot to 1 CD sector (2048 bytes) and append loader
    @truncate -s 2048 $(DIR_ISO_BOOT)/boot.bin
    @cat $(DIR_BUILD)/$(MOD_BOOTLOADER)/loader.bin >> $(DIR_ISO_BOOT)/boot.bin
    @genisoimage -R -J -c boot/bootcat -b boot/boot.bin\
                -no-emul-boot -boot-load-size 4\
                -boot-info-table -o $(ISO_FILE_CD)\
                $(DIR_ISO) &>/dev/null
				
# <Old content down here>

{% endhighlight %}

Before anything else, we check the sizes of the binary files to see if they are within bounds. For stage 1 (`cdboot.bin`) this is one sector of 512 bytes, while for stage 2 (`loader.bin`) this is 31 CD sectors of 2048 bytes (63488 bytes in total). We don't put an `@` before the command so we can see the command that is run before an error occurs. The `-c` passed to `wc` means that the amount of bytes should be counted. The expression within back ticks will be evaluated first and will be substituted with its output: the amount of bytes in the file. The `test` command is passed the `-le` flag, meaning it will check if the first operand is less than or equal to the second operand.

Next, we populate the directory structure. We copy `cdboot.bin` to `build/iso/boot/boot.bin`. This will be the boot file. The next thing we do is using `truncate` to extend the file to a full CD sector of 2048 bytes. Afterwards we redirect the contents of `loader.bin` to the `boot.bin` in the `ISO_BOOT` directory in `Append` mode. `boot.bin` now contains the first stage as the first sector with the second stage appended to the end.

The image itself is created using the `genisoimage` command. The flags that are passed are as follows:

* `-R`: Generate [`Rock Ridge`](https://en.wikipedia.org/wiki/ISO_9660#Rock_Ridge) Directory entries.
* `-J`: Generate ['Joliet'](https://en.wikipedia.org/wiki/ISO_9660#Joliet) Directory entries.
* `-c boot/bootcat`: Creates a boot catalog file named `bootcat` in the `boot` directory of the ISO image.
* `-b boot/boot.bin`: Selects `boot/boot.bin` as the boot file.
* `-no-emul-boot`: Disable floppy emulation. We don't intend on booting on systems where only floppies are supported
* `-boot-load-size 4`: Specify that 4 sectors of 512 bytes should be loaded as the boot sector. This is the most widely supported setting for BIOSes.
* `-boot-info-table`: Insert the `Boot Information Table` into the boot file. We use this to get the sector index and file size of the boot file.
* `-o $(ISO_FILE_CD)`: Sets the output file path for the generated ISO image.
* `$(DIR_ISO)`: The final parameter is the path to the root directory to make the image of.
* `&>/dev/null`: While not a parameter, we redirect `stdout` and `stderr` to `/dev/null` effectively silencing the command. Otherwise `genisoimage` would print out a lot of useless information. It would generally be a good idea to keep `stderr` unaffected, but this tool does all its outputting on `stderr`.

## Conclusion
Now we have a basic environment set up and have a working `Stage 1 Bootloader`. The next things that should be done are writing the second stage of the bootloader as well as writing a basic stub kernel so we can know if we loaded it correctly. This will require setting up a `rust` environment suitable for cross-compiling to a non-existing OS. Both starting the `Stage 2 bootloader` (and maybe finishing it), and setting up a suitable `rust` environment for kernel development will be for the next part.

