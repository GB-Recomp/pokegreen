cmake_minimum_required(VERSION 3.16)
project(pokegreen C CXX)

include(CheckIPOSupported)
include(FetchContent)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE MinSizeRel)
endif()
# Default to a small-binary profile: MinSizeRel + section-based dead
# stripping + post-link symbol stripping. Flip GBRECOMP_SHRINK off for
# debugging; bump GBRECOMP_GENERATED_OPT_LEVEL to 3 for max perf.
option(GBRECOMP_SHRINK "Aggressive size shrink (dead-strip + symbol strip)" ON)
if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
        add_compile_options(-O0 -g)
    elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
        add_compile_options(-O3)
    endif()
    if(GBRECOMP_SHRINK)
        add_compile_options(-ffunction-sections -fdata-sections)
        add_link_options(-Wl,--gc-sections)
        if(NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
            add_link_options(-s)
        endif()
    endif()
endif()

# ---------------------------------------------------------------------------
# gb-recompiled runtime
# When consumed by an umbrella compilation (e.g. GB-Recomp/pgbcomp), the
# parent is expected to provide `gbrt`. We detect that and skip the fetch.
# ---------------------------------------------------------------------------
if(NOT TARGET gbrt)
    set(GBRT_REF "main" CACHE STRING "git ref of GB-Recomp/gb-recompiled to use")
    FetchContent_Declare(gb_recompiled
        GIT_REPOSITORY https://github.com/GB-Recomp/gb-recompiled.git
        GIT_TAG ${GBRT_REF}
        GIT_SHALLOW TRUE
    )
    FetchContent_GetProperties(gb_recompiled)
    if(NOT gb_recompiled_POPULATED)
        FetchContent_Populate(gb_recompiled)
    endif()
    add_subdirectory(${gb_recompiled_SOURCE_DIR}/runtime ${CMAKE_BINARY_DIR}/_gbrt_build)
endif()

# ---------------------------------------------------------------------------
# Cart sources
# ---------------------------------------------------------------------------
set(POKEGREEN_SOURCES
    pokegreen_main.c
    pokegreen.c
    pokegreen_rom.c
    pokegreen_dispatch_chunk_0.c
    pokegreen_dispatch_chunk_1.c
    pokegreen_dispatch_chunk_2.c
    pokegreen_funcs_0.c
    pokegreen_funcs_1.c
    pokegreen_funcs_2.c
    pokegreen_funcs_3.c
    pokegreen_funcs_4.c
    pokegreen_funcs_5.c
    pokegreen_funcs_6.c
    pokegreen_funcs_7.c
    pokegreen_funcs_8.c
    pokegreen_funcs_9.c
    pokegreen_funcs_10.c
    pokegreen_funcs_11.c
    pokegreen_funcs_12.c
    pokegreen_funcs_13.c
    pokegreen_funcs_14.c
    pokegreen_funcs_15.c
    pokegreen_funcs_16.c
    pokegreen_funcs_17.c
    pokegreen_funcs_18.c
    pokegreen_funcs_19.c
    pokegreen_funcs_20.c
    pokegreen_funcs_21.c
    pokegreen_funcs_22.c
    pokegreen_funcs_23.c
    pokegreen_funcs_24.c
    pokegreen_funcs_25.c
)

set(GBRECOMP_GENERATED_OPT_LEVEL "1" CACHE STRING "Optimization level for generated ROM source files")
set_property(CACHE GBRECOMP_GENERATED_OPT_LEVEL PROPERTY STRINGS 0 1 2 3)
set_source_files_properties(${POKEGREEN_SOURCES} PROPERTIES COMPILE_OPTIONS "-O${GBRECOMP_GENERATED_OPT_LEVEL}")

# ---------------------------------------------------------------------------
# pokegreen_cart: static library so umbrella compilations can link us in.
# Exposes `pokegreen_main(argc, argv)` for the launcher's g_games[].
# ---------------------------------------------------------------------------
add_library(pokegreen_cart STATIC ${POKEGREEN_SOURCES})
target_include_directories(pokegreen_cart PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_link_libraries(pokegreen_cart PUBLIC gbrt)

# ---------------------------------------------------------------------------
# Standalone launcher executable. Only built when this is the top-level
# CMake project.
# ---------------------------------------------------------------------------
if(CMAKE_PROJECT_NAME STREQUAL "pokegreen")
    add_executable(pokegreen_launcher launcher_main.cpp)
    target_link_libraries(pokegreen_launcher PRIVATE pokegreen_cart gbrt)
    set_target_properties(pokegreen_launcher PROPERTIES OUTPUT_NAME pokegreen)
endif()
