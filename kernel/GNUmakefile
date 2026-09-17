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

# Check if the architecture is supported.
ifeq ($(filter $(ARCH),aarch64 loongarch64 riscv64 x86_64),)
    $(error Architecture $(ARCH) not supported)
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

# Ensure the dependencies have been obtained.
ifneq ($(filter-out clean distclean,$(or $(MAKECMDGOALS),default)),)
    ifeq ($(wildcard .deps-obtained),)
        $(error Please run the ./get-deps script first)
    endif
endif

# Check if CC is Clang.
override CC_IS_CLANG := $(shell ! $(CC) --version 2>/dev/null | grep -q '^Target: '; echo $$?)

# Check if CFLAGS enables LTO, decided by the last -flto or -fno-lto.
override CFLAGS_HAS_LTO := $(shell ! printf '%s\n' $(CFLAGS) | grep -E '^-f(no-)?lto(=|$$)' | tail -n 1 | grep -q '^-flto'; echo $$?)

# Macros to check if a flag is supported, used as $(call MACRO,flag), and
# expanding to 1 or 0. A second argument names another spelling.

# The name of a flag, up to any "=", with the dashes after the first made
# optional, as compilers leave the value out or spell the flag differently.
override FLAG_NAME = -$(subst -,-?,$(patsubst -%,%,$(firstword $(subst =, ,$(1)))))

# Look for a diagnostic naming the flag, in the untranslated C locale. The
# name has to stand on its own, as "-pie" would otherwise be found inside
# a complaint about "-no-pie".
override FLAG_IS_NAMED = grep -qE '(error|warning|note):.*[^[:alnum:]]($(call FLAG_NAME,$(1))$(if $(2),|$(call FLAG_NAME,$(2))))([^-[:alnum:]]|$$)'

# Check if the compiler supports a flag when compiling.
override CC_HAS_COMPILE_FLAG = $(shell LC_ALL=C $(CC) $(CFLAGS) $(1) -c -x c /dev/null -o /dev/null 2>&1 >/dev/null | $(call FLAG_IS_NAMED,$(1),$(2)); echo $$?)

# Check if the compiler supports a flag when linking, linking nothing.
override CC_HAS_LINK_FLAG = $(shell LC_ALL=C $(CC) $(CFLAGS) $(LDFLAGS) $(1) -Wl,--version 2>&1 >/dev/null | $(call FLAG_IS_NAMED,$(1),$(2)); echo $$?)

# Check if the linker supports a flag by searching its help, as older GNU
# ld take unknown flags for different ones (-no-pie for -n -o -pie).
override LD_HAS_FLAG = $(shell ! $(CC) $(CFLAGS) $(LDFLAGS) -Wl,--help 2>/dev/null | grep -qE '(^|[[:space:]])-?$(1)([[:space:],=[]|$$)'; echo $$?)

# Internal C flags that should not be changed by the user.
override CFLAGS += \
    -Wall \
    -Wextra \
    -std=gnu11 \
    -ffreestanding \
    -fno-common \
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
    -MMD \
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

# Make Clang target the freestanding ELF triple of the architecture.
ifeq ($(CC_IS_CLANG),1)
    override CC += \
        -target $(ARCH)-unknown-none-elf
    # Use LLD unless LDFLAGS picks another linker.
    override LDFLAGS := \
        -fuse-ld=lld \
        $(LDFLAGS)
endif

# Check that the compiler works with the given flags, as the checks below
# would otherwise blame a missing feature.
override CC_COMPILE_ERROR := $(shell LC_ALL=C $(CC) $(CFLAGS) -c -x c /dev/null -o /dev/null 2>&1 >/dev/null | grep -i 'error:' | head -n 1)
ifneq ($(CC_COMPILE_ERROR),)
    $(error The compiler does not work with the given CFLAGS: $(CC_COMPILE_ERROR))
endif

ifeq ($(call CC_HAS_COMPILE_FLAG,-fno-stack-clash-protection),1)
    override CFLAGS += \
        -fno-stack-clash-protection
endif

ifeq ($(CC_IS_CLANG),1)
    # Clang hands freestanding links over to another compiler driver on
    # some targets. Look for the linker flags being passed on as they
    # are, which a linker Clang runs itself would be given translated,
    # as the driver it picks need not be named "gcc" at all. The check
    # is made again below, so it runs on each use. The print flag is
    # kept in a variable, as older makes take a "#" in a call for a
    # comment.
    override CLANG_PRINT_COMMANDS := -\#\#\#
    override CC_LINKS_VIA_DRIVER = $(shell ! $(CC) $(CFLAGS) $(LDFLAGS) -Wl,--version $(CLANG_PRINT_COMMANDS) 2>&1 | grep '^ "' | tail -n 1 | grep -q -- '-Wl,'; echo $$?)

    # If it does, link as for the Linux target, where Clang runs the
    # linker itself, and keep that target's configuration files out. The
    # flag is looked for by compiling, as linking is not checked yet.
    ifeq ($(CC_LINKS_VIA_DRIVER),1)
        override LDFLAGS += \
            -target $(ARCH)-linux-gnu
        ifeq ($(call CC_HAS_COMPILE_FLAG,--no-default-config),1)
            override LDFLAGS += \
                --no-default-config
        endif

        # Check that this worked, as the linker flags would otherwise
        # reach a compiler driver that knows nothing of them.
        ifeq ($(CC_LINKS_VIA_DRIVER),1)
            $(error The compiler still hands the link over to another compiler driver)
        endif
    endif
endif

# The same for linking, once Clang has a linker it can run.
override CC_LINK_ERROR := $(shell LC_ALL=C $(CC) $(CFLAGS) $(LDFLAGS) -Wl,--version 2>&1 >/dev/null | grep -i 'error:' | head -n 1)
ifneq ($(CC_LINK_ERROR),)
    $(error The linker does not work with the given LDFLAGS: $(CC_LINK_ERROR))
endif

# Architecture specific internal flags.
ifeq ($(ARCH),x86_64)
    override CFLAGS += \
        -m64 \
        -march=x86-64 \
        -mno-mmx \
        -mno-sse \
        -mno-red-zone \
        -mcmodel=kernel
    # GCC needs this flag for interrupt handlers and Clang below 3.9 has
    # no name for it, where only a long double could reach the x87.
    ifeq ($(call CC_HAS_COMPILE_FLAG,-mno-80387),1)
        override CFLAGS += \
            -mno-80387
    endif
    ifeq ($(call CC_HAS_COMPILE_FLAG,-fcf-protection=none),1)
        override CFLAGS += \
            -fcf-protection=none
    endif
    override LDFLAGS += \
        -Wl,-m,elf_x86_64
    override NASMFLAGS := \
        -f elf64 \
        $(NASMFLAGS)
endif
ifeq ($(ARCH),aarch64)
    override CFLAGS += \
        -mcpu=generic \
        -march=armv8-a+nofp+nosimd
    ifeq ($(call CC_HAS_COMPILE_FLAG,-mno-outline-atomics),1)
        override CFLAGS += \
            -mno-outline-atomics
    endif
    override CFLAGS += \
        -mcmodel=small
    ifeq ($(call CC_HAS_COMPILE_FLAG,-mbranch-protection=none),1)
        override CFLAGS += \
            -mbranch-protection=none
    endif
    override LDFLAGS += \
        -Wl,-m,aarch64elf
    # Only the linker can work around Cortex-A53 erratum 843419, so
    # require one that can.
    ifeq ($(call LD_HAS_FLAG,--fix-cortex-a53-843419),1)
        override LDFLAGS += \
            -Wl,--fix-cortex-a53-843419
    else
        $(error The linker cannot work around Cortex-A53 erratum 843419)
    endif
endif
ifeq ($(ARCH),riscv64)
    # The ABI comes first, as the instruction set is checked against it.
    override CFLAGS += \
        -mabi=lp64
    # Name Zicsr and Zifencei only if the compiler knows them, as older
    # ones have them in the base ISA. The whole compile is checked, as the
    # flag is not always named when an instruction set is rejected.
    override CC_HAS_ZICSR_ZIFENCEI := $(shell ! $(CC) $(CFLAGS) -march=rv64imac_zicsr_zifencei -c -x c /dev/null -o /dev/null 2>/dev/null; echo $$?)
    ifeq ($(CC_HAS_ZICSR_ZIFENCEI),1)
        override CFLAGS += \
            -march=rv64imac_zicsr_zifencei
    else
        override CFLAGS += \
            -march=rv64imac
    endif
    # Clang called this code model "small" before the ISA's own name.
    ifeq ($(call CC_HAS_COMPILE_FLAG,-mcmodel=medlow,-mcode-model),1)
        override CFLAGS += \
            -mcmodel=medlow
    else
        ifeq ($(call CC_HAS_COMPILE_FLAG,-mcmodel=small,-mcode-model),1)
            override CFLAGS += \
                -mcmodel=small
        else
            $(error The compiler has no name for the low code model)
        endif
    endif
    ifeq ($(call CC_HAS_COMPILE_FLAG,-mno-relax),1)
        override CFLAGS += \
            -mno-relax
    endif
    ifeq ($(call CC_HAS_COMPILE_FLAG,-fcf-protection=none),1)
        override CFLAGS += \
            -fcf-protection=none
    endif
    override LDFLAGS += \
        -Wl,-m,elf64lriscv \
        -Wl,--no-relax
endif
ifeq ($(ARCH),loongarch64)
    # -msoft-float, unlike -mfpu=none, also overrides a -mdouble-float or
    # -msingle-float in CFLAGS, which take effect wherever they appear.
    override CFLAGS += \
        -mabi=lp64s \
        -march=loongarch64 \
        -msoft-float
    # Clang 16 takes the soft float flags but still records the double
    # float ABI, which no linker mixes with soft float objects, so read
    # the ABI out of the ELF header it writes. LTO is turned off for the
    # check, as it would write bitcode with no such header instead.
    ifeq ($(CC_IS_CLANG),1)
        override CC_FLOAT_ABI := $(shell LC_ALL=C $(CC) $(CFLAGS) -fno-lto -c -x c /dev/null -o - 2>/dev/null | od -A n -t x1 -j 48 -N 1)
        ifeq ($(filter %1,$(CC_FLOAT_ABI)),)
            $(error The compiler does not record the soft float ABI)
        endif
    endif
    # Clang called this code model "small" before the ISA's own name.
    ifeq ($(call CC_HAS_COMPILE_FLAG,-mcmodel=normal,-mcode-model),1)
        override CFLAGS += \
            -mcmodel=normal
    else
        ifeq ($(call CC_HAS_COMPILE_FLAG,-mcmodel=small,-mcode-model),1)
            override CFLAGS += \
                -mcmodel=small
        else
            $(error The compiler has no name for the normal code model)
        endif
    endif
    ifeq ($(call CC_HAS_COMPILE_FLAG,-mno-relax),1)
        override CFLAGS += \
            -mno-relax
    endif
    # Do not go through the GOT for external symbols. Only GCC is checked,
    # as Clang had its flag before the architecture existed.
    ifeq ($(CC_IS_CLANG),1)
        override CFLAGS += \
            -fdirect-access-external-data
    else
        ifeq ($(call CC_HAS_COMPILE_FLAG,-mdirect-extern-access),1)
            override CFLAGS += \
                -mdirect-extern-access
        endif
    endif
    # Some Clangs do not record the LoongArch ABI in LTO objects, so pass
    # it to LTO if the IR of an empty file does not record it.
    ifeq ($(CC_IS_CLANG),1)
        ifeq ($(CFLAGS_HAS_LTO),1)
            override CC_LTO_HAS_ABI := $(shell ! $(CC) $(CFLAGS) -S -emit-llvm -x c /dev/null -o - 2>/dev/null | grep -q target-abi; echo $$?)

            ifneq ($(CC_LTO_HAS_ABI),1)
                override LDFLAGS += \
                    -Wl,-plugin-opt=-target-abi=lp64s
            endif
        endif
    endif
    override LDFLAGS += \
        -Wl,-m,elf64loongarch \
        -Wl,--no-relax
endif

# Internal linker flags that should not be changed by the user.
override LDFLAGS += \
    -nostdlib \
    -static \
    -Wl,-z,max-page-size=0x1000 \
    -Wl,-z,noexecstack \
    -Wl,--gc-sections \
    -Wl,-T,linker-scripts/$(ARCH).lds

# Some compilers link static PIEs even when asked for -static, so pass
# them -no-pie when linking if they support it.
ifeq ($(call CC_HAS_LINK_FLAG,-no-pie),1)
    override LDFLAGS += \
        -no-pie
endif

# Tell the linker as well, as the compiler does not always pass this on.
ifeq ($(call LD_HAS_FLAG,-no-pie),1)
    override LDFLAGS += \
        -Wl,-no-pie
endif

# Use "find" to glob all *.c, *.S, and *.asm files in the tree
# (except the src/arch/* directories, as those are gonna be added
# in the next step).
override SRCFILES := $(shell find -L src cc-runtime/src -type f ! -path 'src/arch/*' 2>/dev/null | LC_ALL=C sort)
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

# Link rules for the final executable.
bin-$(ARCH)/$(OUTPUT): GNUmakefile linker-scripts/$(ARCH).lds $(OBJ)
	mkdir -p "$(dir $@)"
	$(CC) $(CFLAGS) $(LDFLAGS) $(OBJ) -o $@

# The compiler may emit calls to the memory functions and the compiler
# runtime that LTO does not account for, so never build those with LTO.
obj-$(ARCH)/src/memory.c.o obj-$(ARCH)/cc-runtime/src/cc-runtime.c.o: override CFLAGS += -fno-lto

# Compilation rules for *.c files.
obj-$(ARCH)/%.c.o: %.c GNUmakefile
	mkdir -p "$(dir $@)"
	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@

# Compilation rules for *.S files.
obj-$(ARCH)/%.S.o: %.S GNUmakefile
	mkdir -p "$(dir $@)"
	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@

ifeq ($(ARCH),x86_64)
# Compilation rules for *.asm (nasm) files.
obj-$(ARCH)/%.asm.o: %.asm GNUmakefile
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
	rm -rf bin-* obj-* .deps-obtained .cache compile_commands.json freestanding-c-hdrs cc-runtime limine-protocol

# Install the final built executable to its final on-root location.
.PHONY: install
install: all
	install -d "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)"
	install -m 644 bin-$(ARCH)/$(OUTPUT) "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)/$(OUTPUT)-$(ARCH)"

# Try to undo whatever the "install" target did.
.PHONY: uninstall
uninstall:
	rm -f "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)/$(OUTPUT)-$(ARCH)"
	-rmdir "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)"
