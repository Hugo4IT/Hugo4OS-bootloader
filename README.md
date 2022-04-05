An edited version of [rust-osdev/bootloader](https://github.com/rust-osdev/bootloader) to add SSE/SIMD (super fast accelerated floating point math) support, with probably more stuff to come.

A lot of things like examples, tests and configurations have been removed, as this repo is only intended as a submodule for [Hugo4OS](https://github.com/Hugo4IT/Hugo4OS), not for distributing as a library/crate by itself.

Original README.md:

# bootloader

[![Docs](https://docs.rs/bootloader/badge.svg)](https://docs.rs/bootloader)
[![Build Status](https://github.com/rust-osdev/bootloader/actions/workflows/build.yml/badge.svg)](https://github.com/rust-osdev/bootloader/actions/workflows/build.yml)
[![Join the chat at https://gitter.im/rust-osdev/bootloader](https://badges.gitter.im/rust-osdev/bootloader.svg)](https://gitter.im/rust-osdev/bootloader?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

An experimental x86_64 bootloader that works on both BIOS and UEFI systems. Written in Rust and some inline assembly, buildable on all platforms without additional build-time dependencies (just some `rustup` components).

## Requirements

You need a nightly [Rust](https://www.rust-lang.org) compiler with the `llvm-tools-preview` component, which can be installed through `rustup component add llvm-tools-preview`.

## Usage

See our [documentation](https://docs.rs/bootloader). Note that the `bootimage` crate is no longer used since version 0.10.0.

## Architecture

This project consists of three separate entities:

- A library with the entry point and boot info definitions that kernels can include as a normal cargo dependency.
- BIOS and UEFI binaries that contain the actual bootloader implementation.
- A `builder` binary to simplify the build process of the BIOS and UEFI binaries.

These three entities are currently all combined in a single crate using cargo feature flags. The reason for this is that the kernel and bootloader must use the exact same version of the `BootInfo` struct to prevent undefined behavior (we did not stabilize the boot info format yet, so it might change between versions).

### Build and Boot

The build and boot process works the following way:

- The `builder` binary is a small command line tool that takes the path to the kernel manifest and binary as arguments. Optionally, it allows to override the cargo target and output dirs. It also accepts a `--quiet` switch and allows to only build the BIOS or UEFI binary instead of both.
- After parsing the arguments, the `builder` binary invokes the actual build command for the BIOS/UEFI binaries, which includes the correct `--target` and `--features` arguments (and `-Zbuild-std`). The kernel manifest and binary paths are passed as `KERNEL_MANIFEST` and `KERNEL` environment variables.
- The next step in the build process is the `build.rs` build script. It only does something when building the BIOS/UEFI binaries (indicated by the `binary` feature), otherwise it is a no-op.
  - The script first runs some sanity checks, e.g. the kernel manifest and binary should be specified in env variables and should exist, the correct target triple should be used, and the `llvm-tools` rustup component should be installed. 
  - Then it copies the kernel executable and strips the debug symbols from it to make it smaller. This does not affect the original kernel binary. The stripped binary is then converted to a byte array and provided to the BIOS/UEFI binaries, either as a Rust `static` or through a linker argument.
  - Next, the bootloader configuration is parsed, which can be specified in a `package.metadata.bootloader` table in the kernel manifest file. This requires some custom string parsing since TOML does not support unsigned 64-bit integers. Parse errors are turned into `compile_error!` calls to give nicer error messages.
  - After parsing the configuration, it is written as a Rust struct definition into a new `bootloader_config.rs` file in the cargo `OUT_DIR`. This file is then included by the UEFI/BIOS binaries.
- After the build script, the compilation continues with either the `bin/uefi.rs` or the `bin/bios.rs`:
  - The `bin/uefi.rs` specifies an UEFI entry point function called `efi_main`. It uses the [`uefi`](https://docs.rs/uefi/0.8.0/uefi/) crate to set up a pixel-based framebuffer using the UEFI GOP protocol. Then it exits the UEFI boot services and stores the physical memory map. The final step is to create some page table abstractions and call into `load_and_switch_to_kernel` function that is shared with the BIOS boot code.
  - The `bin/bios.rs` function does not provide a direct entry point. Instead it includes several assembly files (`asm/stage-*.rs`) that implement the CPU initialization (from real mode to long mode), the framebuffer setup (via VESA), and the memory map creation (via a BIOS call). The assembly stages are explained in more detail below. After the assembly stages, the execution jumps to the `bootloader_main` function in `bios.rs`. There we set up some additional identity mapping, translate the memory map and framebuffer into Rust structs, detect the RSDP table, and create some page table abstractions. Then we call into the `load_and_switch_to_kernel` function like the `bin/uefi.rs`.
- The common `load_and_switch_to_kernel` function is defined in `src/binary/mod.rs`. This is also the file that includes the `bootloader_config.rs` generated by the build script. The `load_and_switch_to_kernel` functions performs the following steps:
  - Parse the kernel binary and map it in a new page table. This includes setting up the correct permissions for each page, initializing `.bss` sections, and allocating a stack with guard page. The relevant functions for these steps are `set_up_mappings` and `load_kernel`.
  - Create the `BootInfo` struct, which abstracts over the differences between BIOS and UEFI booting. This step is implemented in the `create_boot_info` function.
  - Do a context switch and jump to the kernel entry point function. This involves identity-mapping the context switch function itself in both the kernel and bootloader page tables to prevent a page fault after switching page tables. This switch step is implemented in the `switch_to_kernel` and `context_switch` functions.
- As a last step after a successful build, the `builder` binary turns the compiled bootloader executable (includes the kernel) into a bootable disk image. For UEFI, this means that a FAT partition and a GPT disk image are created. For BIOS, the `llvm-objcopy` tool is used to convert the `bootloader` executable to a flat binary, as it already contains a basic MBR.

### BIOS Assembly Stages

When you press the power button the computer loads the BIOS from some flash memory stored on the motherboard. The BIOS initializes and self tests the hardware then loads the first 512 bytes into memory from the media device (i.e. the cdrom or floppy disk). If the last two bytes equal 0xAA55 then the BIOS will jump to location 0x7C00 effectively transferring control to the bootloader.

At this point the CPU is running in 16 bit mode, meaning only the 16 bit registers are available. Also since the BIOS only loads the first 512 bytes this means our bootloader code has to stay below that limit, otherwise we’ll hit uninitialised memory! Using [Bios interrupt calls](https://en.wikipedia.org/wiki/BIOS_interrupt_call) the bootloader prints debug information to the screen.

For more information on how to write a bootloader click [here](http://3zanders.co.uk/2017/10/13/writing-a-bootloader/). The assembler files get imported through the [global_asm feature](https://doc.rust-lang.org/unstable-book/library-features/global-asm.html). The assembler syntax definition used is the one llvm uses: [GNU Assembly](http://microelectronics.esa.int/erc32/doc/as.pdf).

The purposes of the individual assembly stages in this project are the following:

- stage_1.s: This stage initializes the stack, enables the A20 line, loads the rest of the bootloader from disk, and jumps to stage_2.
- stage_2.s: This stage sets the target operating mode, loads the kernel from disk,creates an e820 memory map, enters protected mode, and jumps to the third stage.
- stage_3.s: This stage performs some checks on the CPU (cpuid, long mode), sets up an initial page table mapping (identity map the bootloader, map the P4 recursively, map the kernel blob to 4MB), enables paging, switches to long mode, and jumps to stage_4.

## Future Plans

- [ ] Create a `multiboot2` compatible disk image in addition to the BIOS and UEFI disk images. This would make it possible to use it on top of the GRUB bootloader.
- [ ] Rewrite most of the BIOS assembly stages in Rust. This has already started.
- [ ] Instead of linking the kernel bytes directly with the bootloader, use a filesystem (e.g. FAT) and load the kernel as a separate file.
- [ ] Stabilize the boot info format and make it possible to check the version at runtime.
- [ ] Instead of searching the bootloader source in the cargo cache on building, use the upcoming ["artifact dependencies"](https://github.com/rust-lang/cargo/issues/9096) feature of cargo to download the builder binary separately. Requires doing a boot info version check on build time.
- [ ] Transform this "Future Plans" list into issues and a roadmap.

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or
  http://www.apache.org/licenses/LICENSE-2.0)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.
