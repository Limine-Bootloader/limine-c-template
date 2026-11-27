# Nuke built-in rules.
.SUFFIXES:

# Delete the target of a failed recipe.
.DELETE_ON_ERROR:

# This is the name that our final executable will have.
# Change as needed.
override OUTPUT := kernel

# Target architecture to build for. Default to x86_64.
ARCH := x86_64

# Install prefix; /usr/local is a good, standard default pick.
PREFIX := /usr/local

# Check if the architecture is supported, unless only running distclean,
# which does not use it.
ifneq ($(filter-out distclean,$(or $(MAKECMDGOALS),all)),)
    ifeq ($(filter $(ARCH),aarch64 loongarch64 riscv64 x86_64),)
        $(error Architecture $(ARCH) not supported)
    endif
endif

# User controllable toolchain and toolchain prefix.
TOOLCHAIN :=
TOOLCHAIN_PREFIX :=
ifneq ($(TOOLCHAIN),)
    ifeq ($(TOOLCHAIN_PREFIX),)
        TOOLCHAIN_PREFIX := $(TOOLCHAIN)-
    endif
endif

# User controllable C compiler command.
ifneq ($(TOOLCHAIN_PREFIX),)
    CC := $(TOOLCHAIN_PREFIX)gcc
else
    CC := cc
endif

# Defaults overrides for variables if using "llvm" as toolchain.
ifeq ($(TOOLCHAIN),llvm)
    CC := clang
endif

# User controllable C flags.
CFLAGS := -g -O2 -pipe

# User controllable C preprocessor flags. We set none by default.
CPPFLAGS :=

ifeq ($(ARCH),x86_64)
    # User controllable nasm flags.
    NASMFLAGS := -g
endif

# User controllable linker flags. We set none by default.
LDFLAGS :=

# The goals other than cleaning and uninstalling, which need neither the
# dependencies nor a toolchain.
override BUILD_GOALS := $(filter-out clean distclean uninstall,$(or $(MAKECMDGOALS),all))

# Ensure the dependencies have been obtained, and set up the toolchain
# helper, which records the flags as given, before the internal ones.
ifneq ($(BUILD_GOALS),)
    ifeq ($(wildcard .deps-obtained),)
        $(error Please run the ./get-deps script first)
    endif
    override TCH_ARCH := $(ARCH)
    override TCH_CLANG_TARGET := $(ARCH)-unknown-none-elf
    override TCH_SELECTORS := TOOLCHAIN TOOLCHAIN_PREFIX
    override TCH_CROSS_HINT := use TOOLCHAIN=llvm or a cross compiler for $(ARCH) with TOOLCHAIN_PREFIX
    include gmake-toolchain-helper/init.mk
endif

# Internal C flags that should not be changed by the user.
override CFLAGS += \
    -Wall \
    -Wextra \
    -std=gnu11 \
    -ffreestanding \
    -fno-builtin \
    -fno-exceptions \
    -fno-asynchronous-unwind-tables \
    -fno-unwind-tables \
    -fno-common \
    -fno-strict-aliasing \
    -fno-stack-protector \
    -fno-stack-check \
    -fno-PIC \
    -ffunction-sections \
    -fdata-sections

# Internal C preprocessor flags that should not be changed by the user.
override CPPFLAGS := \
    -nostdinc \
    -I src \
    -I limine-protocol/include \
    -isystem freestanding-c-hdrs/include \
    $(CPPFLAGS) \
    -MD \
    -MP

ifeq ($(ARCH),x86_64)
    # Internal nasm flags that should not be changed by the user.
    override NASMFLAGS := \
        $(patsubst -g,-g -F dwarf,$(NASMFLAGS)) \
        -i src/ \
        -Wall
endif

# Always ask for a build ID, naming its style, as compilers and linkers
# differ on both. Comes before LDFLAGS, which can override it.
override LDFLAGS := \
    -Wl,--build-id=sha1 \
    $(LDFLAGS)

# Internal linker flags that should not be changed by the user.
override LDFLAGS += \
    -nostdlib \
    -static \
    -Wl,-z,max-page-size=0x1000 \
    -Wl,-z,noexecstack \
    -Wl,--gc-sections \
    -Wl,-T,linker-scripts/$(ARCH).lds

# Architecture specific internal flags, and the macro that compilers define
# for the architecture, which the toolchain helper checks for.
ifeq ($(ARCH),x86_64)
    override CFLAGS += \
        -m64 \
        -march=x86-64 \
        -mno-mmx \
        -mno-sse \
        -mno-red-zone \
        -mcmodel=kernel
    override LDFLAGS += \
        -Wl,-m,elf_x86_64
    override NASMFLAGS := \
        -f elf64 \
        $(NASMFLAGS)
    override TCH_TARGET_MACRO := __x86_64__
endif
ifeq ($(ARCH),aarch64)
    override CFLAGS += \
        -mcpu=generic \
        -march=armv8-a+nofp+nosimd \
        -mlittle-endian \
        -mcmodel=small
    override LDFLAGS += \
        -Wl,-m,aarch64elf
    override TCH_TARGET_MACRO := __aarch64__
endif
ifeq ($(ARCH),riscv64)
    # The ABI comes first, as the instruction set is checked against it.
    override CFLAGS += \
        -mabi=lp64
    override LDFLAGS += \
        -Wl,-m,elf64lriscv
    override TCH_TARGET_MACRO := __riscv_xlen=64
endif
ifeq ($(ARCH),loongarch64)
    # -msoft-float, unlike -mfpu=none, also overrides a -mdouble-float or
    # -msingle-float in CFLAGS, which take effect wherever they appear.
    override CFLAGS += \
        -mabi=lp64s \
        -march=loongarch64 \
        -msoft-float
    override LDFLAGS += \
        -Wl,-m,elf64loongarch
    override TCH_TARGET_MACRO := __loongarch_grlen=64
endif

# Check the toolchain with the internal flags, which also makes Clang target
# the architecture and link with LLD unless LDFLAGS picks another linker.
ifneq ($(BUILD_GOALS),)
    include gmake-toolchain-helper/check.mk
endif

# Flags that depend on the toolchain. When nothing is to be built, the
# helper is not included: its macros expand to nothing, and TCH_CC_IS_CLANG
# and TCH_CC_LTO are empty.
override CFLAGS += \
    $(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-fno-stack-clash-protection)

# Architecture specific flags that depend on the toolchain.
ifeq ($(ARCH),x86_64)
    override CFLAGS += \
        $(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-fcf-protection=none)
    # GCC needs this flag for interrupt handlers and Clang below 3.9 has
    # no name for it, where only a long double could reach the x87.
    override CFLAGS += \
        $(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-mno-80387)
endif
ifeq ($(ARCH),aarch64)
    override CFLAGS += \
        $(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-mbranch-protection=none)
    # Clang does not take this ABI name.
    ifneq ($(TCH_CC_IS_CLANG),1)
        override CFLAGS += \
            $(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-mabi=lp64)
    endif
    # Only the linker can work around Cortex-A53 erratum 843419, so
    # require one that can.
    override LDFLAGS += \
        $(call TCH_REQUIRE,$(call TCH_LINKER_HAS_FLAG,--fix-cortex-a53-843419),The linker cannot work around Cortex-A53 erratum 843419)
endif
ifeq ($(ARCH),riscv64)
    override CFLAGS += \
        $(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-fcf-protection=none)
    # Name Zicsr and Zifencei only if the compiler knows them, as older
    # ones have them in the base ISA. The whole compile is checked, as the
    # flag is not always named when an instruction set is rejected.
    override CFLAGS += \
        $(or $(call TCH_DRIVER_COMPILES_WITH,CC,-march=rv64imac_zicsr_zifencei),-march=rv64imac)
    # Clang called this code model "small" before the ISA's own name.
    override CFLAGS += \
        $(or $(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-mcmodel=medlow,-mcode-model),$(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-mcmodel=small,-mcode-model))
    # Many cores trap unaligned accesses for the firmware to emulate.
    # Compilers without the flag never make them.
    override CFLAGS += \
        $(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-mstrict-align)
    # Keep linker relaxation away from gp, which nothing sets up. Linkers
    # without the flag only use gp if the script defines __global_pointer$.
    override LDFLAGS += \
        $(call TCH_LINKER_HAS_FLAG,--no-relax-gp)
endif
ifeq ($(ARCH),loongarch64)
    # Clang 16 takes the soft float flags but still records the double
    # float ABI, which no linker mixes with soft float objects, so read
    # the ABI out of the ELF header it writes. LTO is turned off for the
    # check, as it would write bitcode with no such header instead.
    ifeq ($(TCH_CC_IS_CLANG),1)
        override CC_FLOAT_ABI := $(shell $(CC) $(CFLAGS) -fno-lto -c -x c /dev/null -o - 2>/dev/null | od -A n -t x1 -j 48 -N 1)
        ifeq ($(filter %1,$(CC_FLOAT_ABI)),)
            $(error The compiler does not record the soft float ABI)
        endif
    endif
    # Do not go through the GOT for external symbols. Only GCC is checked,
    # as Clang had its flag before the architecture existed.
    ifeq ($(TCH_CC_IS_CLANG),1)
        override CFLAGS += \
            -fdirect-access-external-data
    else
        override CFLAGS += \
            $(call TCH_REQUIRE,$(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-mdirect-extern-access),The compiler cannot keep external data out of the GOT)
    endif
    # Clang called this code model "small" before the ISA's own name.
    override CFLAGS += \
        $(or $(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-mcmodel=normal,-mcode-model),$(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-mcmodel=small,-mcode-model))
    # Some Clangs do not record the LoongArch ABI in LTO objects, so pass
    # it to LTO if the IR of an empty file does not record it.
    ifeq ($(TCH_CC_IS_CLANG),1)
        ifeq ($(TCH_CC_LTO),1)
            override CC_LTO_HAS_ABI := $(shell ! $(CC) $(CFLAGS) -S -emit-llvm -x c /dev/null -o - 2>/dev/null | grep -q target-abi; echo $$?)
            ifneq ($(CC_LTO_HAS_ABI),1)
                override LDFLAGS += \
                    -Wl,-plugin-opt=-target-abi=lp64s
            endif
        endif
    endif
endif

# Some compilers link static PIEs even when asked for -static, so pass
# them -no-pie when linking if they support it, and tell the linker as
# well, as the compiler does not always pass this on.
override LDFLAGS += \
    $(call TCH_DRIVER_HAS_LINK_FLAG,-no-pie)
override LDFLAGS += \
    $(call TCH_LINKER_HAS_FLAG,-no-pie)

# Use "find" to glob all *.c, *.S, and *.asm files in the tree
# (except the src/arch/* directories, as those are gonna be added
# in the next step).
override SRCFILES := $(shell find -L src freestanding-lgrt/src -type f ! -path 'src/arch/*' 2>/dev/null | LC_ALL=C sort)
# Add architecture specific files, if they exist.
override SRCFILES += $(shell find -L src/arch/$(ARCH) -type f 2>/dev/null | LC_ALL=C sort)
# Obtain the object and header dependencies file names.
override CFILES := $(filter %.c,$(SRCFILES))
override ASFILES := $(filter %.S,$(SRCFILES))
override OBJ := $(addprefix obj-$(ARCH)/,$(CFILES:.c=.c.o) $(ASFILES:.S=.S.o))
override HEADER_DEPS := $(addprefix obj-$(ARCH)/,$(CFILES:.c=.c.d) $(ASFILES:.S=.S.d))
ifeq ($(ARCH),x86_64)
override NASMFILES := $(filter %.asm,$(SRCFILES))
override OBJ += $(addprefix obj-$(ARCH)/,$(NASMFILES:.asm=.asm.o))
override HEADER_DEPS += $(addprefix obj-$(ARCH)/,$(NASMFILES:.asm=.asm.d))
endif

# Default target. This must come first, before header dependencies.
.PHONY: all
all: bin-$(ARCH)/$(OUTPUT)

# Include header dependencies.
-include $(HEADER_DEPS)

# Each kind of file depends on records of the variables it is built with,
# named after them, which the toolchain helper remakes when they change.
override TCH_RECORDS := $(addprefix obj-$(ARCH)/,CC CFLAGS CPPFLAGS LDFLAGS)
ifeq ($(ARCH),x86_64)
override TCH_RECORDS += obj-$(ARCH)/NASMFLAGS
endif
ifneq ($(BUILD_GOALS),)
    include gmake-toolchain-helper/records.mk
endif

# Link rules for the final executable.
bin-$(ARCH)/$(OUTPUT): GNUmakefile obj-$(ARCH)/CC obj-$(ARCH)/CFLAGS obj-$(ARCH)/LDFLAGS linker-scripts/$(ARCH).lds $(OBJ)
	mkdir -p "$(dir $@)"
	$(CC) $(CFLAGS) $(LDFLAGS) $(OBJ) -o $@

# The compiler may emit calls to the memory functions and the compiler
# runtime that LTO does not account for, so never build those with LTO.
obj-$(ARCH)/src/memory.c.o obj-$(ARCH)/freestanding-lgrt/src/freestanding-lgrt.c.o: override CFLAGS += -fno-lto

# Compilation rules for *.c files.
obj-$(ARCH)/%.c.o: %.c GNUmakefile obj-$(ARCH)/CC obj-$(ARCH)/CFLAGS obj-$(ARCH)/CPPFLAGS
	mkdir -p "$(dir $@)"
	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@

# Compilation rules for *.S files.
obj-$(ARCH)/%.S.o: %.S GNUmakefile obj-$(ARCH)/CC obj-$(ARCH)/CFLAGS obj-$(ARCH)/CPPFLAGS
	mkdir -p "$(dir $@)"
	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@

ifeq ($(ARCH),x86_64)
# Compilation rules for *.asm (nasm) files.
obj-$(ARCH)/%.asm.o: %.asm GNUmakefile obj-$(ARCH)/NASMFLAGS
	mkdir -p "$(dir $@)"
	nasm $(NASMFLAGS) -MD $(@:.o=.d) -MP $< -o $@
endif

# Remove object files and the final executable.
.PHONY: clean
clean:
	rm -rf bin-$(ARCH) obj-$(ARCH)

# Remove everything built and generated including downloaded dependencies.
.PHONY: distclean
distclean:
	rm -rf bin-* obj-* .deps-obtained .cache compile_commands.json freestanding-c-hdrs freestanding-lgrt gmake-toolchain-helper limine-protocol

# Install the final built executable to its final on-root location. Not
# install(1), which takes other arguments on illumos and Solaris.
.PHONY: install
install: all
	mkdir -p "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)"
	cp bin-$(ARCH)/$(OUTPUT) "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)/$(OUTPUT)-$(ARCH)"
	chmod 644 "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)/$(OUTPUT)-$(ARCH)"

# Try to undo whatever the "install" target did.
.PHONY: uninstall
uninstall:
	rm -f "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)/$(OUTPUT)-$(ARCH)"
	rmdir "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)" 2>/dev/null || :
