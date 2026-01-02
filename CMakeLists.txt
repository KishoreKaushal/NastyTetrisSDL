cmake_minimum_required(VERSION 3.25.1)

project(nasty_tetris_sdl)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# ------------------ SDL ------------------
# point CMake to the folder containing 'SDL3Config.cmake'
# so that it searches inside the subfolders like 'cmake/' automatically
list(APPEND CMAKE_PREFIX_PATH "${CMAKE_CURRENT_SOURCE_DIR}/vendored/SDL3")
find_package(SDL3 REQUIRED)
# --------------- End: SDL ----------------


# Linking a pre-compiled library which doesn't have a CMake config.
# ------------------ SDL3_mixer ------------------
# Define where SDL_mixer is located relative to this CMakeLists.txt
set(MIXER_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/vendored/SDL_mixer")
set(MIXER_BUILD "${MIXER_ROOT}/build")
# import our custom build for SDL3_mixer. Start by creating a "fake" library that points to our files
add_library(SDL3_mixer::SDL3_mixer SHARED IMPORTED)


if(WIN32)
    # point to the .lib file (used during linking) and .dll (for reference)
    set_target_properties(SDL3_mixer::SDL3_mixer PROPERTIES
            IMPORTED_IMPLIB "${MIXER_BUILD}/SDL3_mixer.lib"
            IMPORTED_LOCATION "${MIXER_BUILD}/SDL3_mixer.dll"
    )
elseif (APPLE)
    set_target_properties(SDL3_mixer::SDL3_mixer PROPERTIES
            IMPORTED_LOCATION "${MIXER_BUILD}/libSDL3_mixer.dylib"
    )
else () # LINUX
    set_target_properties(SDL3_mixer::SDL3_mixer PROPERTIES
            IMPORTED_LOCATION "${MIXER_BUILD}/libSDL3_mixer.so"
    )
endif ()

# include directories are common
target_include_directories(SDL3_mixer::SDL3_mixer INTERFACE "${MIXER_ROOT}/include")

# INTERFACE
# means that SDL3_mixer don't need the header for compiling itself (as it is already compiled)
# rather it is exposing to public so that external program using this library (eg., our current project)
# knows how to use the SDL3_mixer.
# Can't use PRIVATE because then CMake would ignore them because it isn't compiling the DLL for SDL3_mixer
# as it is already compiled. Consequently, no external program (our project) would be able to use it.
# PUBLIC = INTERFACE + PRIVATE. and we know that PRIVATE part is irrelavant because it's already compiled.
# ----------------- End: SDL3_mixer --------------


# --- Building this project ---
add_executable(${PROJECT_NAME} src/main.cpp)

# --- link everything ---
target_link_libraries(${PROJECT_NAME} PRIVATE
        SDL3::SDL3-shared
        SDL3_mixer::SDL3_mixer
)

# Set the output directory for 'my_program'
set_target_properties(${PROJECT_NAME} PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/dist"
)

# --- auto-copy DLL (critical) for distributing the game
# WHY? Because windows required the DLL to be next to the .exe to run
if (WIN32)
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${MIXER_BUILD}/SDL3_mixer.dll"
            $<TARGET_FILE_DIR:${PROJECT_NAME}>
            COMMENT "Copying SDL3_mixer.dll to output directory..."
    )
endif ()
# no need to do this on Linux and Mac
# because of a concept called RPATH (Run-Path).
# When you compile a `game` on Linux/Mac, the linker secretly writes a "note" inside the executable file
# that says:
# i need libSDL3_mixer.so. Please look for it at this location: /Users/kshr4/...

# because of this embedded note, you can run the game from your build folder, and it will
# automatically reach out and find the library where it was built. Hence no need to copy the file next to
# executable.

# EXCEPTION: "Distribution"
# If you send your game to a friend, the RPATH pointing to /user/kshr4/... will fail on their computer.
# Fo on linux .. you typically set the RPATH to $ORIGIN => means look in the same folder as me.
# so in this case, yes, you would copy the .so file next to the binary, just like Windows.

# On Mac: You typically build an App Bundle (.app) This is a special folder structure.
# Exes go in: Game.app/Contents/MacOS/
# DyLibs go in: Game.app/Contents/Frameworks/
# Cmake handles this bundle creation automatically if you configure it correctly (using MACOSX_BUNDLE)


# ==============================================================================
# CROSS-PLATFORM DISTRIBUTION LOGIC
# ==============================================================================

if (WIN32 AND CMAKE_BUILD_TYPE STREQUAL "Release")
    message(STATUS "Release build detected. Configuring 'dist' folder generation.")

    # 1. Define a clean output dir
    set(DIST_DIR "${CMAKE_BINARY_DIR}/dist")

    # 2. List all the DLLs that you need
    set(REQUIRED_DLLS
            "${CMAKE_CURRENT_SOURCE_DIR}/vendored/SDL3/lib/x64/SDL3.dll"
            "${MIXER_BUILD}/SDL3_mixer.dll"
    )

    # 3. Create the Post-Build Command
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
            # copy all DLLs to dist/
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${REQUIRED_DLLS}
            "${DIST_DIR}"
            COMMENT "Windows: Copying DLLs to ${DIST_DIR}"
    )
endif()