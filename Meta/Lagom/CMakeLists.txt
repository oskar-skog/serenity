cmake_minimum_required (VERSION 3.21)

project(
    Lagom
    VERSION 0.0.0
    DESCRIPTION "Host build of SerenityOS libraries and applications"
    HOMEPAGE_URL "https://github.com/SerenityOS/serenity"
    LANGUAGES C CXX
)

if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS "12")
  message(FATAL_ERROR
      "A GCC version less than 12 was detected (${CMAKE_CXX_COMPILER_VERSION}), this is unsupported.\n"
      "Please re-read the build instructions documentation, and upgrade your host compiler.\n")
endif()

# This is required for CMake (when invoked for a Lagom-only build) to
# ignore any files downloading during the build, e.g. UnicodeData.txt.
# https://cmake.org/cmake/help/latest/policy/CMP0058.html
cmake_policy(SET CMP0058 NEW)

# Make CMAKE_EXE_LINKER_FLAGS have an effect on `try_compile()` jobs.
# This is required if we want to have the `LAGOM_USE_LINKER` option
# take effect in `check_linker_flag` checks.
cmake_policy(SET CMP0056 NEW)

get_filename_component(
    SERENITY_PROJECT_ROOT "${PROJECT_SOURCE_DIR}/../.."
    ABSOLUTE CACHE
)
set(SerenityOS_SOURCE_DIR "${SERENITY_PROJECT_ROOT}" CACHE STRING "")

list(APPEND CMAKE_MODULE_PATH "${SERENITY_PROJECT_ROOT}/Meta/CMake")

if(NOT COMMAND serenity_option)
    macro(serenity_option)
        set(${ARGV})
    endmacro()
endif()

include(check_for_dependencies)
include(use_linker)
include(lagom_options NO_POLICY_SCOPE)

if(ENABLE_ALL_THE_DEBUG_MACROS)
    include(all_the_debug_macros)
endif()

# FIXME: Is it worth inventing `serenity_dependent_option` ?
if (ENABLE_LAGOM_LADYBIRD)
    set(ENABLE_LAGOM_LIBWEB ON CACHE BOOL "" FORCE)
endif()

# FIXME: BUILD_SHARED_LIBS has a default of OFF, as it's intended to be set by the
#        user when configuring the project. We should instead change libjs-test262
#        and oss-fuzz to set this option on their end, and enable it by default in
#        Meta/serenity.sh. This is #9867.
option(BUILD_SHARED_LIBS "Build shared libraries instead of static libraries" ON)

find_package(Threads REQUIRED)
# FIXME: This global link libraries is required to workaround linker issues (on some systems)
# from the Ladybird import. See https://github.com/SerenityOS/serenity/issues/16847
link_libraries(Threads::Threads)

if (ENABLE_LAGOM_CCACHE)
    include(setup_ccache)
endif()

if (ENABLE_FUZZERS_LIBFUZZER OR ENABLE_FUZZERS_OSSFUZZ)
	set(ENABLE_FUZZERS ON)
endif()

# We need to make sure not to build code generators for Fuzzer builds, as they already have their own main.cpp
# Instead, we import them from a previous install of Lagom. This mandates a two-stage build for fuzzers.
# The same concern goes for cross-compile builds, where we need the tools built for the host
set(BUILD_LAGOM_TOOLS ON)
if (ENABLE_FUZZERS OR CMAKE_CROSSCOMPILING)
    find_package(LagomTools REQUIRED)
    set(BUILD_LAGOM_TOOLS OFF)
endif()

include(flac_spec_tests)
include(ca_certificates_data)
include(lagom_compile_options)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if (EMSCRIPTEN)
    set(CMAKE_EXECUTABLE_SUFFIX ".js")
    add_compile_options(-gsource-map)
    add_link_options(--emrun "SHELL:-s ALLOW_MEMORY_GROWTH")
endif()

if (ENABLE_ADDRESS_SANITIZER)
    add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
    set(LINKER_FLAGS "${LINKER_FLAGS} -fsanitize=address")
endif()

if (ENABLE_MEMORY_SANITIZER)
    add_compile_options(-fsanitize=memory -fsanitize-memory-track-origins -fno-omit-frame-pointer)
    set(LINKER_FLAGS "${LINKER_FLAGS} -fsanitize=memory -fsanitize-memory-track-origins")
endif()

if (ENABLE_UNDEFINED_SANITIZER)
    add_compile_options(-fsanitize=undefined -fno-omit-frame-pointer)
    if (UNDEFINED_BEHAVIOR_IS_FATAL)
        add_compile_options(-fno-sanitize-recover=undefined)
    endif()
    if (APPLE AND CMAKE_CXX_COMPILER_ID MATCHES "Clang$" AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL "17")
        add_compile_options(-fno-sanitize=function)
    endif()
    set(LINKER_FLAGS "${LINKER_FLAGS} -fsanitize=undefined")
endif()

if (ENABLE_COMPILETIME_FORMAT_CHECK)
    add_compile_definitions(ENABLE_COMPILETIME_FORMAT_CHECK)
endif()

if (HAIKU)
    # Haiku needs some extra compile definitions to make important stuff in its headers available.
    add_compile_definitions(_DEFAULT_SOURCE)
    add_compile_definitions(_GNU_SOURCE)
    add_compile_definitions(__USE_GNU)
endif()

if (ENABLE_FUZZERS)
    add_compile_options(-fno-omit-frame-pointer)
endif()

if (ENABLE_LAGOM_LADYBIRD AND (ENABLE_FUZZERS OR ENABLE_COMPILER_EXPLORER_BUILD))
    message(FATAL_ERROR
        "Ladybird build not supported for Fuzzers or Compiler Explorer."
        "Disable ENABLE_LAGOM_LADYBIRD and try again."
    )
endif()

CHECK_INCLUDE_FILE(pulse/pulseaudio.h HAVE_PULSEAUDIO)

add_library(JSClangPlugin INTERFACE)
add_library(GenericClangPlugin INTERFACE)

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang$")
    add_compile_options(-Wno-overloaded-virtual)
    # FIXME: Re-enable this check when the warning stops triggering, or document why we can't stop it from triggering.
    # For now, there is a lot of unused private fields in LibWeb that trigger this that could be removed.
    # See issue #14137 for details
    add_compile_options(-Wno-unused-private-field)

    if (ENABLE_FUZZERS_LIBFUZZER)
        add_compile_options(-fsanitize=fuzzer)
        set(LINKER_FLAGS "${LINKER_FLAGS} -fsanitize=fuzzer")
    endif()

    # Vanilla host builds only for building the clang plugins
    if (ENABLE_CLANG_PLUGINS AND NOT CROSS_COMPILING AND NOT ENABLE_FUZZERS AND NOT ENABLE_COMPILER_EXPLORER_BUILD)
        add_subdirectory(ClangPlugins)
        depend_on_clang_plugin(JSClangPlugin LibJSGCClangPlugin)
        depend_on_clang_plugin(GenericClangPlugin LambdaCaptureClangPlugin)
    endif()
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    if (ENABLE_FUZZERS_LIBFUZZER)
        message(FATAL_ERROR
            "Fuzzer Sanitizer (-fsanitize=fuzzer) is only supported for Fuzzer targets with LLVM. "
            "Reconfigure CMake with -DCMAKE_C_COMPILER and -DCMAKE_CXX_COMPILER pointing to a clang-based toolchain "
	    "or build binaries without built-in fuzzing support by setting -DENABLE_FUZZERS instead."
        )
    endif()
endif()

# These are here to support Fuzzili builds further down the directory stack
set(ORIGINAL_CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS}")
set(ORIGINAL_CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS}")
set(ORIGINAL_CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS}")

set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${LINKER_FLAGS}")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${LINKER_FLAGS}")
set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} ${LINKER_FLAGS}")

configure_file(../../AK/Debug.h.in AK/Debug.h @ONLY)

include_directories(../../)
include_directories(../../Userland/)
include_directories(../../Userland/Libraries/)
include_directories(../../Userland/Services)
include_directories(${CMAKE_BINARY_DIR})
include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(${CMAKE_CURRENT_BINARY_DIR}/Userland/Libraries)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/Userland/Services)

# install rules, think about moving to its own helper cmake file
include(CMakePackageConfigHelpers)

# find_package(<package>) call for consumers to find this project
set(package Lagom CACHE STRING "")

# Allow package maintainers to freely override the path for the configs
set(Lagom_INSTALL_CMAKEDIR "${CMAKE_INSTALL_DATADIR}/${package}"
    CACHE PATH "CMake package config location relative to the install prefix")
mark_as_advanced(Lagom_INSTALL_CMAKEDIR)

install(
    FILES "${SERENITY_PROJECT_ROOT}/Meta/CMake/lagom-install-config.cmake"
    DESTINATION "${Lagom_INSTALL_CMAKEDIR}"
    RENAME "${package}Config.cmake"
    COMPONENT Lagom_Development
)

install(
    EXPORT LagomTargets
    NAMESPACE Lagom::
    DESTINATION "${Lagom_INSTALL_CMAKEDIR}"
    COMPONENT Lagom_Development
)

function(lagom_lib target_name fs_name)
    cmake_parse_arguments(LAGOM_LIBRARY "" "LIBRARY_TYPE" "SOURCES;LIBS" ${ARGN})
    string(REPLACE "Lib" "" library ${target_name})
    if (NOT LAGOM_LIBRARY_LIBRARY_TYPE)
        set(LAGOM_LIBRARY_LIBRARY_TYPE "")
    endif()
    add_library(${target_name} ${LAGOM_LIBRARY_LIBRARY_TYPE} ${LAGOM_LIBRARY_SOURCES})
    set_target_properties(
        ${target_name} PROPERTIES
        VERSION "${PROJECT_VERSION}"
        SOVERSION "${PROJECT_VERSION_MAJOR}"
        EXPORT_NAME ${library}
        OUTPUT_NAME lagom-${fs_name}
    )
    target_link_libraries(${target_name} PRIVATE ${LAGOM_LIBRARY_LIBS})
    target_link_libraries(${target_name} PUBLIC GenericClangPlugin)

    if (NOT "${target_name}" STREQUAL "AK")
        target_link_libraries(${target_name} PRIVATE AK)
    endif()

    # FIXME: Clean these up so that we don't need so many include dirs
    target_include_directories(${target_name} INTERFACE
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/Services>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/Userland/Libraries>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/Userland/Services>
    )
    add_lagom_library_install_rules(${target_name} ALIAS_NAME ${library})
    # FIXME: Move this to serenity_install_headers
    install(
        DIRECTORY "${SERENITY_PROJECT_ROOT}/Userland/Libraries/Lib${library}"
        COMPONENT Lagom_Development
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
        FILES_MATCHING PATTERN "*.h"
    )
    serenity_generated_sources(${target_name})
endfunction()

function(lagom_test source)
    cmake_parse_arguments(LAGOM_TEST "" "NAME;WORKING_DIRECTORY" "LIBS" ${ARGN})
    if (NOT DEFINED LAGOM_TEST_NAME)
        get_filename_component(LAGOM_TEST_NAME ${source} NAME_WE)
    endif()
    add_executable(${LAGOM_TEST_NAME} ${source})
    target_link_libraries(${LAGOM_TEST_NAME} PRIVATE AK LibCore LibFileSystem LibTest LibTestMain ${LAGOM_TEST_LIBS})
    add_test(
        NAME ${LAGOM_TEST_NAME}
        COMMAND ${LAGOM_TEST_NAME}
        WORKING_DIRECTORY ${LAGOM_TEST_WORKING_DIRECTORY}
    )
    set_target_properties(${LAGOM_TEST_NAME} PROPERTIES LAGOM_WORKING_DIRECTORY "${LAGOM_TEST_WORKING_DIRECTORY}")
endfunction()

function(lagom_add_executable name)
    # if any of the sources are .jakt files, switch to using serenity_jakt_executable()
    # and pass the non-jakt sources as extra cpp sources.
    set(jakt_source)
    set(cpp_sources)
    foreach(source ${ARGN})
        if (${source} MATCHES "\\.jakt$")
            if (jakt_source)
                message(FATAL_ERROR "Multiple main jakt files in executable ${name}")
            else()
                set(jakt_source ${source})
            endif()
        else()
            list(APPEND cpp_sources ${source})
        endif()
    endforeach()

    if (jakt_source)
        if (ENABLE_JAKT AND NOT EMSCRIPTEN)
            serenity_jakt_executable(${name} MAIN_SOURCE ${jakt_source} CPP_SOURCES ${cpp_sources})
        else()
            message(STATUS "Skipping ${name} because jakt is disabled")
            # Make an empty c++ file:
            file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/${name}.cpp" "// This file was generated by CMake to skip a jakt executable\nint main() {}")
            add_executable(${name} "${CMAKE_CURRENT_BINARY_DIR}/${name}.cpp")
            set_target_properties(${name} PROPERTIES SERENITY_SKIP_BUILD TRUE)
        endif()
    else()
        add_executable(${name} ${cpp_sources})
    endif()
endfunction()

function(lagom_utility name)
    cmake_parse_arguments(LAGOM_UTILITY "" "" "SOURCES;LIBS" ${ARGN})

    lagom_add_executable("${name}" ${LAGOM_UTILITY_SOURCES})
    target_link_libraries("${name}" PRIVATE AK LibCore ${LAGOM_UTILITY_LIBS})
endfunction()

function(serenity_test test_src sub_dir)
    cmake_parse_arguments(PARSE_ARGV 2 SERENITY_TEST "MAIN_ALREADY_DEFINED" "CUSTOM_MAIN" "LIBS")
    # FIXME: Pass MAIN_ALREADY_DEFINED and CUSTOM_MAIN to support tests that use them.
    lagom_test(${test_src} LIBS ${SERENITY_TEST_LIBS} WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
endfunction()

function(serenity_bin name)
    lagom_add_executable(${name} ${SOURCES} ${GENERATED_SOURCES})
    add_executable(Lagom::${name} ALIAS ${name})
    target_link_libraries(${name} PUBLIC GenericClangPlugin)
    install(
        TARGETS ${target_name}
        EXPORT LagomTargets
        RUNTIME #
            COMPONENT Lagom_Runtime
        LIBRARY #
            COMPONENT Lagom_Runtime
            NAMELINK_COMPONENT Lagom_Development
        ARCHIVE #
            COMPONENT Lagom_Development
        INCLUDES #
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )
    serenity_generated_sources(${name})
endfunction()

function(serenity_lib name fs_name)
    cmake_parse_arguments(PARSE_ARGV 2 SERENITY_LIB "" "TYPE" "")
    lagom_lib(${name} ${fs_name} LIBRARY_TYPE ${SERENITY_LIB_TYPE} SOURCES ${SOURCES} ${GENERATED_SOURCES})
endfunction()

function(serenity_install_headers dir)
endfunction()

function(serenity_install_sources dir)
endfunction()

macro(add_serenity_subdirectory path)
    add_subdirectory("${SERENITY_PROJECT_ROOT}/${path}" "${CMAKE_CURRENT_BINARY_DIR}/${path}")
endmacro()

option(BUILD_EVERYTHING "Build all optional components" ON)

if (NOT TARGET all_generated)
    # Meta target to run all code-gen steps in the build.
    add_custom_target(all_generated)
endif()

# Plugins need to be installed in order to be used by non-lagom builds
add_lagom_library_install_rules(GenericClangPlugin)
add_lagom_library_install_rules(JSClangPlugin)

# Create mostly empty targets for system libraries we don't need to build for Lagom
add_library(LibC INTERFACE)
add_library(LibCrypt INTERFACE)
add_library(LibSystem INTERFACE)
if (NOT APPLE AND NOT ANDROID AND NOT EMSCRIPTEN AND NOT ${CMAKE_SYSTEM_NAME} MATCHES "OpenBSD" AND NOT HAIKU)
    target_link_libraries(LibCrypt INTERFACE crypt) # LibCore::Account uses crypt() but it's not in libcrypt on macOS
endif()
if (SERENITYOS)
    # Serenity only allows syscalls from LibSystem, so if someone asks for that on Lagom,
    # we need to pass that through to the system's LibSystem.
    target_link_libraries(LibSystem INTERFACE system)
endif()
add_library(NoCoverage INTERFACE)
# "install" these special targets to placate CMake
install(TARGETS LibC LibCrypt LibSystem NoCoverage EXPORT LagomTargets)

# AK
add_serenity_subdirectory(AK)
lagom_lib(AK ak SOURCES ${AK_SOURCES})
find_package(Backtrace)
if (Backtrace_FOUND AND NOT "${Backtrace_LIBRARIES}" STREQUAL "")
    target_link_libraries(AK PRIVATE ${Backtrace_LIBRARIES})
    target_include_directories(AK PRIVATE ${Backtrace_INCLUDE_DIRS})
endif()

# LibCoreMinimal
add_serenity_subdirectory(Userland/Libraries/LibCore)
if (${CMAKE_SYSTEM_NAME} MATCHES "NetBSD")
    # NetBSD has its shm_open and shm_unlink functions in librt so we need to link that
    target_link_libraries(LibCoreMinimal PRIVATE rt)
endif()

# LibMain
add_serenity_subdirectory(Userland/Libraries/LibMain)

# LibFileSystem
# This is needed even if Lagom is not enabled because it is depended upon by code generators.
add_serenity_subdirectory(Userland/Libraries/LibFileSystem)

# LibTimeZone
# This is needed even if Lagom is not enabled because it is depended upon by code generators.
add_serenity_subdirectory(Userland/Libraries/LibTimeZone)
# We need an install rule for LibTimeZone b/c it is a manual OBJECT library instead of serenity_lib
install(TARGETS LibTimeZone EXPORT LagomTargets)

# LibIDL
# This is used by the BindingsGenerator so needs to always be built.
add_serenity_subdirectory(Userland/Libraries/LibIDL)

# LibGUI - only GML
# This is used by the GML compiler and therefore always needed.
set(LIBGUI_GML_SOURCES
    GML/Lexer.cpp
    GML/Parser.cpp
)
list(TRANSFORM LIBGUI_GML_SOURCES PREPEND "${SERENITY_PROJECT_ROOT}/Userland/Libraries/LibGUI/")
lagom_lib(LibGUI_GML gui_gml
    SOURCES ${LIBGUI_GML_SOURCES}
)

# LibELF
# This is used by the PrekernelPEImageGenerator and therefore always needed.
add_serenity_subdirectory(Userland/Libraries/LibELF)

# Manually install AK headers
install(
    DIRECTORY "${SERENITY_PROJECT_ROOT}/AK"
    COMPONENT Lagom_Development
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    FILES_MATCHING PATTERN "*.h"
)
install(FILES
    ${Lagom_BINARY_DIR}/AK/Debug.h
    COMPONENT Lagom_Development
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/AK"
)

# Code Generators and other host tools
if (BUILD_LAGOM_TOOLS)
    add_subdirectory(Tools)
endif()

if (BUILD_LAGOM)

    # Create lagom equivalent of Serenity's Build/foo/Root/res (which for Serenity becomes /res in the disk image).
    # FIXME: Add more files as needed.
    set(LAGOM_RES "${Lagom_BINARY_DIR}/Root/res")
    file(MAKE_DIRECTORY "${LAGOM_RES}/fonts")
    foreach(font
        LiberationMono-Bold.ttf
        LiberationMono-BoldItalic.ttf
        LiberationMono-Italic.ttf
        LiberationMono-Regular.ttf
        LiberationSans-Bold.ttf
        LiberationSans-BoldItalic.ttf
        LiberationSans-Italic.ttf
        LiberationSans-Regular.ttf
        LiberationSerif-Bold.ttf
        LiberationSerif-BoldItalic.ttf
        LiberationSerif-Italic.ttf
        LiberationSerif-Regular.ttf
        )
        configure_file("${SERENITY_PROJECT_ROOT}/Base/res/fonts/${font}" "${LAGOM_RES}/fonts/${font}" COPYONLY)
    endforeach()

    # Lagom Libraries
    set(lagom_standard_libraries
        AccelGfx
        Archive
        Audio
        Compress
        Crypto
        Cpp
        DeviceTree
        DNS
        Diff
        Gemini
        Gfx
        GL
        GLSL
        GPU
        HID
        HTTP
        IMAP
        ImageDecoderClient
        IPC
        JIT
        JS
        Line
        Locale
        Manual
        Markdown
        PDF
        Media
        Protocol
        Regex
        RIFF
        SoftGPU
        Shell
        SQL
        Syntax
        TextCodec
        Threading
        TLS
        Unicode
        URL
        Wasm
        WebSocket
        XML
    )

    target_link_libraries(LibCore PRIVATE LibURL Threads::Threads)
    if (${CMAKE_SYSTEM_NAME} MATCHES "NetBSD")
        # NetBSD has its shm_open and shm_unlink functions in librt so we need to link that
        target_link_libraries(LibCore PRIVATE rt)
    endif()
    if (${CMAKE_SYSTEM_NAME} MATCHES "SunOS")
        # Solaris has socket and networking related functions in two extra libraries
        target_link_libraries(LibCore PRIVATE nsl socket)
    endif()
    if (HAIKU)
        # Haiku has networking related functions in the network library
        target_link_libraries(LibCore PRIVATE network)
    endif()

    # These are needed for both LibWeb and LibProtocol.
    compile_ipc(${SERENITY_PROJECT_ROOT}/Userland/Services/RequestServer/RequestClient.ipc Userland/Services/RequestServer/RequestClientEndpoint.h)
    compile_ipc(${SERENITY_PROJECT_ROOT}/Userland/Services/RequestServer/RequestServer.ipc Userland/Services/RequestServer/RequestServerEndpoint.h)

    if (ENABLE_LAGOM_LIBWEB)
        list(APPEND lagom_standard_libraries Web WebView)

        compile_ipc(${SERENITY_PROJECT_ROOT}/Userland/Services/WebContent/WebContentServer.ipc Userland/Services/WebContent/WebContentServerEndpoint.h)
        compile_ipc(${SERENITY_PROJECT_ROOT}/Userland/Services/WebContent/WebContentClient.ipc Userland/Services/WebContent/WebContentClientEndpoint.h)
        compile_ipc(${SERENITY_PROJECT_ROOT}/Userland/Services/WebContent/WebDriverClient.ipc Userland/Services/WebContent/WebDriverClientEndpoint.h)
        compile_ipc(${SERENITY_PROJECT_ROOT}/Userland/Services/WebContent/WebDriverServer.ipc Userland/Services/WebContent/WebDriverServerEndpoint.h)
    endif()

    if (NOT EMSCRIPTEN)
        list(APPEND lagom_standard_libraries Disassembly)
    endif()

    foreach(lib IN LISTS lagom_standard_libraries)
        add_serenity_subdirectory("Userland/Libraries/Lib${lib}")
    endforeach()

    # GUI
    set(LIBGUI_SOURCES
        GML/Lexer.cpp
        GML/Parser.cpp
        GML/SyntaxHighlighter.cpp
        Icon.cpp
        Model.cpp
        ModelIndex.cpp
    )
    list(TRANSFORM LIBGUI_SOURCES PREPEND "${SERENITY_PROJECT_ROOT}/Userland/Libraries/LibGUI/")
    lagom_lib(LibGUI gui
        SOURCES ${LIBGUI_SOURCES}
        LIBS LibCore LibGfx LibSyntax)

    # FIXME: How about we don't include Kernel/API from random high-level libraries?
    install(FILES ${SERENITY_PROJECT_ROOT}/Kernel/API/KeyCode.h DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/Kernel/API")

    target_link_libraries(LibLocale PRIVATE LibTimeZone)

    add_serenity_subdirectory(Userland/Shell)

    if (NOT ENABLE_FUZZERS AND NOT ENABLE_COMPILER_EXPLORER_BUILD AND NOT ANDROID AND NOT IOS)
        # Lagom Services
        add_serenity_subdirectory(Userland/Services)

        # Lagom Utilities
        lagom_utility(abench SOURCES ../../Userland/Utilities/abench.cpp LIBS LibMain LibFileSystem LibAudio)
        lagom_utility(aconv SOURCES ../../Userland/Utilities/aconv.cpp LIBS LibMain LibFileSystem LibAudio)

        if (NOT EMSCRIPTEN)
            lagom_utility(adjtime SOURCES ../../Userland/Utilities/adjtime.cpp LIBS LibMain)
        endif()

        lagom_utility(animation SOURCES ../../Userland/Utilities/animation.cpp LIBS LibGfx LibMain)
        lagom_utility(base64 SOURCES ../../Userland/Utilities/base64.cpp LIBS LibMain)

        if (NOT EMSCRIPTEN)
            lagom_utility(disasm SOURCES ../../Userland/Utilities/disasm.cpp LIBS LibELF LibDisassembly LibMain)
            lagom_utility(file SOURCES ../../Userland/Utilities/file.cpp LIBS LibAudio LibArchive LibCompress LibELF LibGfx LibIPC LibMain)
        endif()

        lagom_utility(gml-format SOURCES ../../Userland/Utilities/gml-format.cpp LIBS LibGUI LibMain)
        lagom_utility(gzip SOURCES ../../Userland/Utilities/gzip.cpp LIBS LibCompress LibMain)

        # Work around bug in JetBrains distributed CMake 3.27.2 where this causes infinite recursion in
        # export_components() when called from CLion Nova by checking if we already have Ladybird included
        # This technically breaks people including Lagom as a submodule and wanting Ladybird targets, but... who? why?
        if (ENABLE_LAGOM_LADYBIRD AND PROJECT_IS_TOP_LEVEL)
            add_serenity_subdirectory(Ladybird)
        endif()

        if (APPLE)
            add_serenity_subdirectory(Meta/Lagom/Contrib/MacPDF)
        endif()

        find_package(SDL2 QUIET)
        if (SDL2_FOUND)
            add_serenity_subdirectory(Meta/Lagom/Contrib/VideoPlayerSDL)
        endif()

        lagom_utility(icc SOURCES ../../Userland/Utilities/icc.cpp LIBS LibGfx LibMain LibURL)
        lagom_utility(iconv SOURCES ../../Userland/Utilities/iconv.cpp LIBS LibTextCodec LibMain)
        lagom_utility(image SOURCES ../../Userland/Utilities/image.cpp LIBS LibGfx LibMain)
        lagom_utility(imgcmp SOURCES ../../Userland/Utilities/imgcmp.cpp LIBS LibGfx LibMain)
        lagom_utility(isobmff SOURCES ../../Userland/Utilities/isobmff.cpp LIBS LibGfx LibMain)
        lagom_utility(ttfdisasm SOURCES ../../Userland/Utilities/ttfdisasm.cpp LIBS LibGfx LibMain)
        lagom_utility(js SOURCES ../../Userland/Utilities/js.cpp LIBS LibCrypto LibJS LibLine LibLocale LibMain LibTextCodec Threads::Threads)
        lagom_utility(hello SOURCES ../../Userland/Utilities/hello-world.jakt)

        if (EMSCRIPTEN)
            lagom_utility(libjs SOURCES Wasm/js_repl.cpp LIBS LibJS LibLocale LibTimeZone LibUnicode)
            target_link_options(libjs PRIVATE
                    -sEXPORTED_FUNCTIONS=_initialize_repl,_execute,_free
                    -sEXPORTED_RUNTIME_METHODS=stringToNewUTF8
                    -sERROR_ON_UNDEFINED_SYMBOLS=0
                    -sENVIRONMENT=web)
        endif()

        lagom_utility(lzcat SOURCES ../../Userland/Utilities/lzcat.cpp LIBS LibCompress LibMain)
        lagom_utility(markdown-check SOURCES ../../Userland/Utilities/markdown-check.cpp LIBS LibFileSystem LibMarkdown LibMain LibManual LibURL)
        lagom_utility(mkfs.fat SOURCES  ../../Userland/Utilities/mkfs.fat.cpp LIBS LibFileSystem LibMain)

        if (NOT EMSCRIPTEN)
            lagom_utility(ntpquery SOURCES ../../Userland/Utilities/ntpquery.cpp LIBS LibMain)
        endif()

        lagom_utility(pdf SOURCES ../../Userland/Utilities/pdf.cpp LIBS LibGfx LibPDF LibMain)
        lagom_utility(sed SOURCES ../../Userland/Utilities/sed.cpp LIBS LibFileSystem LibMain LibRegex)
        lagom_utility(sql SOURCES ../../Userland/Utilities/sql.cpp LIBS LibFileSystem LibIPC LibLine LibMain LibSQL)
        lagom_utility(tar SOURCES ../../Userland/Utilities/tar.cpp LIBS LibArchive LibCompress LibFileSystem LibMain)
        lagom_utility(test262-runner SOURCES ../../Tests/LibJS/test262-runner.cpp LIBS LibJS LibFileSystem)
        lagom_utility(unzip SOURCES ../../Userland/Utilities/unzip.cpp LIBS LibArchive LibCompress LibCrypto LibFileSystem LibMain)

        if (CMAKE_SYSTEM_NAME STREQUAL "Linux")
            include(CheckCSourceCompiles)
            # Check for musl's declaration of __assert_fail
            check_c_source_compiles(
                "
                #include <assert.h>
                __attribute__((__noreturn__)) void __assert_fail(char const* assertion, char const* file, int line, char const* function) {}
                int main() {}
                "
                ASSERT_FAIL_HAS_INT
            )
        endif()

        if (ASSERT_FAIL_HAS_INT OR EMSCRIPTEN)
            target_compile_definitions(test262-runner PRIVATE ASSERT_FAIL_HAS_INT)
        endif()

        lagom_utility(wasm SOURCES ../../Userland/Utilities/wasm.cpp LIBS LibFileSystem LibWasm LibLine LibMain LibJS)
        lagom_utility(xml SOURCES ../../Userland/Utilities/xml.cpp LIBS LibFileSystem LibMain LibXML LibURL)
        lagom_utility(xzcat SOURCES ../../Userland/Utilities/xzcat.cpp LIBS LibCompress LibMain)
        lagom_utility(fdtdump SOURCES ../../Userland/Utilities/fdtdump.cpp LIBS LibDeviceTree LibMain)
        lagom_utility(hiddump SOURCES ../../Userland/Utilities/hiddump.cpp LIBS LibHID LibMain)
        lagom_utility(crypto-bench SOURCES ../../Userland/Utilities/crypto-bench.cpp LIBS LibMain LibCrypto)

        enable_testing()
        # LibTest
        set(LIBTEST_SOURCES
            ../../Userland/Libraries/LibTest/AsyncTestStreams.cpp
            ../../Userland/Libraries/LibTest/TestSuite.cpp
            ../../Userland/Libraries/LibTest/CrashTest.cpp
        )
        add_library(LibTest ${LIBTEST_SOURCES})
        target_link_libraries(LibTest PRIVATE AK LibCore LibFileSystem)
        set_target_properties(LibTest PROPERTIES OUTPUT_NAME lagom-test)
        add_library(
            LibTestMain
            OBJECT
            "${SERENITY_PROJECT_ROOT}/Userland/Libraries/LibTest/TestMain.cpp"
        )
        target_link_libraries(LibTest PUBLIC GenericClangPlugin)
        target_link_libraries(LibTestMain PUBLIC GenericClangPlugin)

        # LibTest tests from Tests/
        set(TEST_DIRECTORIES
            AK
            LibCrypto
            LibCompress
            LibGL
            LibGfx
            LibHID
            LibHTTP
            LibIMAP
            LibLocale
            LibMarkdown
            LibPDF
            LibSQL
            LibMedia
        LibTest
            LibTextCodec
            LibTTF
            LibTimeZone
            LibUnicode
            LibURL
                LibXML
        )
        if (ENABLE_LAGOM_LIBWEB)
            list(APPEND TEST_DIRECTORIES LibWeb)
            list(APPEND TEST_DIRECTORIES LibWebView)
        endif()
        if (LINUX)
            list(APPEND TEST_DIRECTORIES LibELF)
        endif()
        if (ENABLE_CLANG_PLUGINS AND CMAKE_CXX_COMPILER_ID MATCHES "Clang$")
            list(APPEND TEST_DIRECTORIES ClangPlugins)
        endif()

        foreach (dir IN LISTS TEST_DIRECTORIES)
            add_serenity_subdirectory("Tests/${dir}")
        endforeach()

        # LibTLS needs a special working directory to find cacert.pem
        lagom_test(../../Tests/LibTLS/TestTLSHandshake.cpp LibTLS LIBS LibTLS LibCrypto)
        lagom_test(../../Tests/LibTLS/TestTLSCertificateParser.cpp LibTLS LIBS LibTLS LibCrypto)

        # The FLAC tests need a special working directory to find the test files
        lagom_test(../../Tests/LibAudio/TestFLACSpec.cpp LIBS LibAudio WORKING_DIRECTORY "${FLAC_TEST_PATH}/..")

        lagom_test(../../Tests/LibAudio/TestPlaybackStream.cpp LIBS LibAudio)
        if (HAVE_PULSEAUDIO)
            target_compile_definitions(TestPlaybackStream PRIVATE HAVE_PULSEAUDIO=1)
        endif()

        # LibCore
        lagom_test(../../Tests/LibCore/TestLibCoreArgsParser.cpp)

        if ((LINUX OR APPLE) AND NOT EMSCRIPTEN)
            lagom_test(../../Tests/LibCore/TestLibCoreFileWatcher.cpp)
            lagom_test(../../Tests/LibCore/TestLibCorePromise.cpp LIBS LibThreading)
        endif()

        lagom_test(../../Tests/LibCore/TestLibCoreDateTime.cpp LIBS LibTimeZone)

        # RegexLibC test POSIX <regex.h> and contains many Serenity extensions
        # It is therefore not reasonable to run it on Lagom, and we only run the Regex test
        lagom_test(../../Tests/LibRegex/Regex.cpp LIBS LibRegex WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/../../Tests/LibRegex)

        # test-jpeg-roundtrip
        add_executable(test-jpeg-roundtrip
            ../../Userland/Utilities/test-jpeg-roundtrip.cpp)
        target_link_libraries(test-jpeg-roundtrip AK LibGfx LibMain)

        # JavaScriptTestRunner + LibTest tests
        # test-js
        add_executable(test-js
            ../../Tests/LibJS/test-js.cpp
            ../../Userland/Libraries/LibTest/JavaScriptTestRunnerMain.cpp)
        target_link_libraries(test-js AK LibCore LibFileSystem LibTest LibJS)
        add_test(
            NAME JS
            COMMAND test-js --show-progress=false
        )
        set_tests_properties(JS PROPERTIES ENVIRONMENT SERENITY_SOURCE_DIR=${SERENITY_PROJECT_ROOT})

        # Extra tests from Tests/LibJS
        lagom_test(../../Tests/LibJS/test-invalid-unicode-js.cpp LIBS LibJS)
        lagom_test(../../Tests/LibJS/test-value-js.cpp LIBS LibJS)

        # Spreadsheet
        add_executable(test-spreadsheet
                ../../Tests/Spreadsheet/test-spreadsheet.cpp
                ../../Userland/Libraries/LibTest/JavaScriptTestRunnerMain.cpp)
        target_link_libraries(test-spreadsheet AK LibCore LibFileSystem LibTest LibJS)
        add_test(
                NAME Spreadsheet
                COMMAND test-spreadsheet --show-progress=false
        )
        set_tests_properties(Spreadsheet PROPERTIES ENVIRONMENT SERENITY_SOURCE_DIR=${SERENITY_PROJECT_ROOT})

        # test-wasm
        add_executable(test-wasm
            ../../Tests/LibWasm/test-wasm.cpp
            ../../Userland/Libraries/LibTest/JavaScriptTestRunnerMain.cpp)
        target_link_libraries(test-wasm AK LibCore LibFileSystem LibTest LibWasm LibJS LibCrypto)
        add_test(
            NAME WasmParser
            COMMAND test-wasm --show-progress=false ${CMAKE_CURRENT_BINARY_DIR}/Userland/Libraries/LibWasm/Tests
        )
        set_tests_properties(WasmParser PROPERTIES
            SKIP_RETURN_CODE 1
            ENVIRONMENT SERENITY_SOURCE_DIR=${SERENITY_PROJECT_ROOT}
        )

        # Tests that are not LibTest based
        # Shell
        set(SHELL_TESTS
            backgrounding.sh
            brace-exp.sh
            builtin-redir.sh
            builtin-test.sh
            control-structure-as-command.sh
            function.sh
            heredocs.sh
            if.sh
            immediate.sh
            loop.sh
            match.sh
            sigpipe.sh
            slice.sh
            special-vars.sh
            subshell.sh
            valid.sh
        )
        foreach(TEST_SCRIPT ${SHELL_TESTS})
            get_filename_component(TEST_NAME ${TEST_SCRIPT} NAME_WE)
            add_test(
                NAME "Shell-${TEST_NAME}"
                COMMAND Shell --skip-shellrc "${TEST_SCRIPT}"
                WORKING_DIRECTORY ${SERENITY_PROJECT_ROOT}/Tests/LibShell/Tests
            )
            set_tests_properties("Shell-${TEST_NAME}" PROPERTIES
                TIMEOUT 10
                FAIL_REGULAR_EXPRESSION "FAIL"
                PASS_REGULAR_EXPRESSION "PASS"
            )
        endforeach()

        # FIXME: When we are using CMake >= 3.21, the library installations can be replaced with RUNTIME_DEPENDENCIES.
        #        https://cmake.org/cmake/help/latest/command/install.html
        include(get_linked_lagom_libraries.cmake)
        get_linked_lagom_libraries(js js_libraries)

        install(TARGETS js ${js_libraries} COMPONENT js)

        set(CPACK_GENERATOR "TGZ")
        set(CPACK_STRIP_FILES TRUE)
        set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
        set(CPACK_COMPONENTS_ALL js)
        if (APPLE)
            if("arm64" IN_LIST CMAKE_OSX_ARCHITECTURES AND "x86_64" IN_LIST CMAKE_OSX_ARCHITECTURES)
                set(CPACK_SYSTEM_NAME "macOS-universal2")
            else()
                set(CPACK_SYSTEM_NAME "macOS-${CMAKE_SYSTEM_PROCESSOR}")
            endif()
        else()
            set(CPACK_SYSTEM_NAME "${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR}")
        endif()

        set(CPACK_ARCHIVE_JS_FILE_NAME "serenity-js-${CPACK_SYSTEM_NAME}")
        set(CPACK_PACKAGE_FILE_NAME "serenity-js-${CPACK_SYSTEM_NAME}")
        include(CPack)
    endif()
endif()

if (ENABLE_FUZZERS)
    add_subdirectory(Fuzzers)
else()
    export_components("${CMAKE_BINARY_DIR}/components.ini")
endif()

if (NOT "$ENV{LAGOM_TARGET}" STREQUAL "")
    add_custom_target(run-lagom-target
        COMMAND "${CMAKE_COMMAND}"
            -E env "SERENITY_SOURCE_DIR=${SERENITY_PROJECT_ROOT}" "$<TARGET_FILE:$ENV{LAGOM_TARGET}>"
            $ENV{LAGOM_ARGS}

        WORKING_DIRECTORY "$<TARGET_PROPERTY:$ENV{LAGOM_TARGET},LAGOM_WORKING_DIRECTORY>"
        DEPENDS "$<TARGET_FILE:$ENV{LAGOM_TARGET}>"
        USES_TERMINAL
        VERBATIM
    )
endif()
