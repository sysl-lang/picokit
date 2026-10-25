cmake_minimum_required(VERSION 3.13)

# **The carrier board takes either a Pico or a Pico W**: everything it hangs off the GPIOs is the
# RP2040's, and nothing the kit does goes near the wireless chip. What the build is pinned to is the
# `sh.sysl.pico` package, whose `init` starts stdio and then the CYW43 — so `pico_w` is what this
# links today. `README.md § A plain Pico` says what the other one wants.
set(PICO_BOARD pico_w CACHE STRING "Board type")

# **The RP2040 is Armv6-M and there is no second personality to choose**, which is where this differs
# from the same file in `ogol-pico2`: an RP2350 carries Cortex-M33s and Hazard3 RISC-V cores and the
# CMake there has to pick. Here there is one core family and one sysl target.
#
# `sysl build-c` is a program CMake knows nothing about, so it has to be told: getting this wrong is
# not a link error but an image that faults at whatever executes first.
set(SYSL_TARGET thumbv6m-freestanding)

include(pico_sdk_import.cmake)

project(picokit C CXX ASM)

pico_sdk_init()

set(SYSL_ARCHIVE ${CMAKE_CURRENT_BINARY_DIR}/libpicokit.a)

# **`sh.sysl.pico` carries C, and it needs the SDK's headers to compile it.** `sysl build-c` is
# outside CMake's world, so what it can see has to be handed to it.
#
# The libc half comes from the toolchain rather than being written down: `-print-sysroot` answers with
# the directory whose `include` holds newlib's headers, and the obvious path one level up has no
# `string.h` in it.
execute_process(
    COMMAND ${CMAKE_C_COMPILER} -print-sysroot
    OUTPUT_VARIABLE ARM_SYSROOT
    OUTPUT_STRIP_TRAILING_WHITESPACE)

# The SDK half is asked of the SDK, because it generates some of its headers into its own build
# directory and a fixed list written here would go stale the first time it moved one. What is asked is
# the **executable's** resolved list rather than any one library's, since `cyw43.h` includes
# `lwip/netif.h` two link libraries away. Naming a target defined further down is fine: a generator
# expression is resolved after the whole file is read.
set(SYSL_C_INCLUDES
    ${ARM_SYSROOT}/include
    $<TARGET_PROPERTY:picokit,INCLUDE_DIRECTORIES>)

# And the macros, which finding the headers does not supply: `pico/cyw43_arch.h` `#error`s rather than
# picking an architecture variant, so every include path can be right and the compile still stops
# inside a header.
set(SYSL_C_DEFINES $<TARGET_PROPERTY:picokit,COMPILE_DEFINITIONS>)

add_custom_command(
    OUTPUT ${SYSL_ARCHIVE} ${SYSL_ARCHIVE}.h
    COMMAND sysl build-c ${CMAKE_CURRENT_SOURCE_DIR}/picokit --target ${SYSL_TARGET}
            "$<$<BOOL:${SYSL_C_INCLUDES}>:--include-path;$<JOIN:${SYSL_C_INCLUDES},;--include-path;>>"
            "$<$<BOOL:${SYSL_C_DEFINES}>:-D;$<JOIN:${SYSL_C_DEFINES},;-D;>>"
            -o ${SYSL_ARCHIVE}
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/picokit/main.sysl
            ${CMAKE_CURRENT_SOURCE_DIR}/picokit/sysl.hocon
    COMMENT "Compiling the Pico Breadboard Kit for ${SYSL_TARGET}"
    # **`COMMAND_EXPAND_LISTS`, without which the two generator expressions above are useless.** A
    # quoted argument that expands to a `;`-separated list stays *one* argument unless this is given,
    # so `sysl` is handed a single `--include-path;/a;--include-path;/b` and answers `Unknown option`.
    # Unquoting is not the fix either — CMake then splits the variable at parse time and the `$<`
    # never evaluates at all.
    COMMAND_EXPAND_LISTS
    VERBATIM)

add_custom_target(sysl_archive DEPENDS ${SYSL_ARCHIVE})

# There is no C in this project at all: sysl exports `main`, the SDK's `crt0` branches straight into
# it, and the archive in the source list is what satisfies CMake's "a target must have sources" rule
# with no translation unit to infer a language from.
add_executable(picokit ${SYSL_ARCHIVE})

set_target_properties(picokit PROPERTIES LINKER_LANGUAGE C)

add_dependencies(picokit sysl_archive)

# **`pico_cyw43_arch_none` is what `sh.sysl.pico`'s `init` resolves against, and it is a Pico W
# target.** A plain Pico does not fail here — it fails at the link, on `cyw43_arch_init`, because an
# `extern` this program calls has nothing to point at. `README.md` says what a plain Pico wants
# instead; it is a change to the program rather than to this file.
target_link_libraries(picokit pico_stdlib pico_cyw43_arch_none ${SYSL_ARCHIVE})

pico_enable_stdio_usb(picokit 1)
pico_enable_stdio_uart(picokit 0)

pico_add_extra_outputs(picokit)
