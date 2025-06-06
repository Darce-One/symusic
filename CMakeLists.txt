cmake_minimum_required(VERSION 3.20)
project(symusic)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_OSX_DEPLOYMENT_TARGET 11.0)

option(BUILD_SYMUSIC_PY "Build the python binding of symusic" OFF)
option(BUILD_SYMUSIC_EXAMPLE "Build the example target" OFF)
option(BUILD_SYMUSIC_TEST "Build the test target" OFF)
option(ENABLE_LTO "Enables link-time optimization, requires compiler support." ON)
option(VTUNE "Compile Options for Intel VTune" Off)
option(MEM_LEAK_WARNING "Enable memory leak warning" OFF)
option(ENABLE_COVERAGE "Enable code coverage" OFF)

# Enable code coverage if requested
if(ENABLE_COVERAGE)
    message(STATUS "Code coverage enabled")
    if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -fprofile-arcs -ftest-coverage")
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
    else()
        message(WARNING "Code coverage is only supported with GCC and Clang")
    endif()
endif()

# set /utf-8 for MSVC
if(MSVC)
    add_compile_options(-bigobj)
    add_compile_options(/utf-8)
endif()

# Add multilingual path test executable
add_executable(test_multilang_path tests/cpp/test_multilang_path.cpp src/io/common.cpp)
target_include_directories(test_multilang_path PRIVATE include)
target_link_libraries(test_multilang_path symusic)

if(MEM_LEAK_WARNING)
    message(STATUS "Nanobind memory leak warning ENABLED")
    add_definitions(-DMEM_LEAK_WARNING)
else ()
    message(STATUS "Nanobind memory leak warning DISABLED")
endif()

add_subdirectory(3rdparty/fmt EXCLUDE_FROM_ALL)
add_subdirectory(3rdparty/minimidi EXCLUDE_FROM_ALL)
add_subdirectory(3rdparty/prestosynth EXCLUDE_FROM_ALL)
add_subdirectory(3rdparty/pyvec EXCLUDE_FROM_ALL)

include_directories(include)
include_directories(3rdparty/pdqsort)
include_directories(3rdparty/zpp_bits)
include_directories(3rdparty)

file(GLOB_RECURSE symusic_src src/*.cpp src/io/*.cpp)

foreach(src_file ${symusic_src})
    message(STATUS "symusic_src: ${src_file}")
endforeach()

add_library(symusic ${symusic_src})
target_link_libraries(symusic fmt::fmt-header-only)
target_link_libraries(symusic minimidi)
target_link_libraries(symusic prestosynth)
target_link_libraries(symusic pyvec)

target_include_directories(symusic PUBLIC include)
target_include_directories(symusic PUBLIC 3rdparty/pdqsort)
target_include_directories(symusic PUBLIC 3rdparty/zpp_bits)
target_include_directories(symusic PUBLIC 3rdparty)


if(BUILD_SYMUSIC_PY)
    message("Building python binding.")
    find_package(Python REQUIRED COMPONENTS Interpreter Development.Module)
    # set symusic target to O3 and release mode because nanobind use minisize
#   execute_process(
#        COMMAND "${Python_EXECUTABLE}" -m nanobind --cmake_dir
#        OUTPUT_STRIP_TRAILING_WHITESPACE OUTPUT_VARIABLE NB_DIR)
#        list(APPEND CMAKE_PREFIX_PATH "${NB_DIR}")
#   find_package(nanobind CONFIG REQUIRED)
    # show python executable path
    message(STATUS "Python_EXECUTABLE: ${Python_EXECUTABLE}")

    add_subdirectory(3rdparty/nanobind EXCLUDE_FROM_ALL)

    nanobind_add_module(
        core
        NB_STATIC STABLE_ABI LTO PROTECT_STACK NOSTRIP
        NB_DOMAIN symusic
        py_src/core.cpp py_src/py_utils.h py_src/bind_vector_copy.h
    )
    target_link_libraries(core PRIVATE symusic)

    install(TARGETS core LIBRARY DESTINATION symusic)
#   install abc2midi and midi2abc
    add_subdirectory(3rdparty/abcmidi)
    install(TARGETS abc2midi midi2abc DESTINATION symusic/bin)
endif()

if(BUILD_SYMUSIC_EXAMPLE)
    add_subdirectory(./3rdparty/nanobench EXCLUDE_FROM_ALL)

    add_executable(note_count examples/cpp/note_count.cpp)
    add_executable(dump examples/cpp/dump.cpp)
    add_executable(io_bench examples/cpp/io_bench.cpp)
    add_executable(synth examples/cpp/synth.cpp)
    add_executable(adjust_time examples/cpp/adjust_time.cpp)
    add_executable(process_midi_directory examples/cpp/process_midi_directory.cpp)

    target_link_libraries(note_count PRIVATE symusic)
    target_link_libraries(dump PRIVATE symusic)
    target_link_libraries(io_bench PRIVATE symusic nanobench)
    target_link_libraries(synth PUBLIC symusic)
    target_link_libraries(adjust_time PUBLIC symusic)
    target_link_libraries(process_midi_directory PRIVATE symusic)
endif()

if(BUILD_SYMUSIC_TEST)
    Include(FetchContent)
    FetchContent_Declare(
            Catch2
            GIT_REPOSITORY https://github.com/catchorg/Catch2.git
            GIT_TAG v3.4.0 # or a later release
    )
    FetchContent_MakeAvailable(Catch2)
    list(APPEND CMAKE_MODULE_PATH ${catch2_SOURCE_DIR}/extras)
    add_executable(symusic_test tests/cpp/test_main.cpp)
    target_include_directories(symusic_test PRIVATE tests/cpp)
    target_link_libraries(symusic_test PRIVATE Catch2::Catch2WithMain symusic)
    include(CTest)
    include(Catch)
    enable_testing()
    add_test(
        NAME symusic_ctest
        COMMAND $<TARGET_FILE:symusic_test> --success
    )
endif()


if (ENABLE_LTO)
    include(CheckIPOSupported)
    check_ipo_supported(RESULT has_lto OUTPUT lto_check_output)
    if(has_lto)
        if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
            message(STATUS "Clang detected, use -fuse-ld=lld")
            set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fuse-ld=lld")
        endif ()
        message(STATUS "Link-time optimization (LTO) enabled")
        set_property(TARGET symusic PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
        if(BUILD_SYMUSIC_PY)
            set_property(TARGET core PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
        endif ()
    else()
        message(WARNING "Link-time optimization (LTO) is not supported: \n${lto_check_output}")
    endif()
endif()

# enable parallel compilation for MSVC
if(MSVC)
    message(STATUS "MSVC parallel compilation enabled")
    add_compile_options($<$<CXX_COMPILER_ID:MSVC>:/Gm->)
    cmake_host_system_information(RESULT CPU_NUMBER_OF_LOGICAL_CORES QUERY NUMBER_OF_LOGICAL_CORES)
    add_compile_options($<$<CXX_COMPILER_ID:MSVC>:/MP${CPU_NUMBER_OF_LOGICAL_CORES}>)
endif()

if(VTUNE)
    message(STATUS "VTune enabled")
    if(MSVC)
        add_compile_options(/Zi)
        add_compile_options(/MDd)
        add_compile_options(/D "TBB_USE_THREADING_TOOLS")
    else()
        add_compile_options(-g)
    endif ()
endif ()

set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CMAKE_COMMAND} -E time")
set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK "${CMAKE_COMMAND} -E time")
