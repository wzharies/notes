```cmake
#
# Stole from: https://github.com/cmu-db/noisepage
#


#######################################################################################################################
# Project definition.
#######################################################################################################################

cmake_minimum_required(VERSION 2.8)

# must be added before project() or enable_language() command
# set(CMAKE_CXX_COMPILER "/usr/bin/g++-4.8")

project(
    dolphindb
    LANGUAGES CXX
)


#######################################################################################################################
# Check compiler: only support g++-4.8.5 now
#######################################################################################################################

message(STATUS "Compiler: ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION}")

if (NOT "${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU")
    message(FATAL_ERROR "${PROJECT_NAME} only support g++ now, but your compiler is ${CMAKE_CXX_COMPILER_ID}.")
endif ()

if (NOT CMAKE_CXX_COMPILER_VERSION VERSION_EQUAL 4.8.5)
    message(FATAL_ERROR "${PROJECT_NAME} requires g++ 4.8.5, but your g++ version is ${CMAKE_CXX_COMPILER_VERSION}.")
endif ()


#######################################################################################################################
# Safety checks: run CMake from a build subdirectory.
#######################################################################################################################

# People keep running CMake in the wrong folder, completely nuking their project or creating weird bugs.
# This checks if you're running CMake from a folder that already has CMakeLists.txt.
# Importantly, this catches the common case of running it from the root directory.
file(TO_CMAKE_PATH "${PROJECT_BINARY_DIR}/CMakeLists.txt" PATH_TO_CMAKELISTS_TXT)
if (EXISTS "${PATH_TO_CMAKELISTS_TXT}")
    message(FATAL_ERROR "Run CMake from a build subdirectory! \"mkdir build ; cd build ; cmake ..\" \
    Some junk files were created in this folder (CMakeCache.txt, CMakeFiles); you should delete those.")
endif ()


#######################################################################################################################
# ???
#######################################################################################################################

find_program(CCACHE_PROGRAM ccache)
if (CCACHE_PROGRAM)
    message("Found ccache! ${CCACHE_PROGRAM}")
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ${CCACHE_PROGRAM})
    set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ${CCACHE_PROGRAM})
endif (CCACHE_PROGRAM)


#######################################################################################################################
# CMake options and global variables.
#
# CMake build types, specify with -DCMAKE_BUILD_TYPE={option}.
#   Debug (default), Release, RelWithDebInfo.
#
# CMake options, specify with -DDDB_{option}=On.
#   DDB_USE_ASAN                      : Enable ASAN, a fast memory error detector. Default OFF.
#   DDB_USE_TCMALLOC                  : Link with tcmalloc instead of system malloc. Default ON.
#   DDB_USE_JIT                       : Build the JIT version of DolphinDB. Default OFF.
#
# CMake global variables. These are NOT CMake options, i.e., these variables are internal.
#   BUILD_SUPPORT_DIR             : Helper scripts for building belongs here.
#   BUILD_SUPPORT_DATA_DIR        : Helper data for building belongs here.
#   DDB_COMPILE_OPTIONS     : Compile options to be added to DolphinDB.
#   DDB_INCLUDE_DIRECTORIES : Include directories to be used for DolphinDB.
#   DDB_LINK_LIBRARIES      : Link libraries to be added to DolphinDB.
#   DDB_LINK_OPTIONS        : Link options to be added to DolphinDB.
#######################################################################################################################

# Default to DEBUG builds if -DCMAKE_BUILD_TYPE was not specified.
if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Debug)
endif (NOT CMAKE_BUILD_TYPE)

option(DDB_USE_ASAN
        "Enable ASAN, a fast memory error detector. https://clang.llvm.org/docs/AddressSanitizer.html"
        OFF)

option(DDB_USE_TCMALLOC
        "Link with tcmalloc instead of system malloc"
        ON)

option(DDB_USE_JIT
        "Build the JIT version of DolphinDB"
        OFF)

set(BUILD_SUPPORT_DIR "${CMAKE_SOURCE_DIR}/build-support")
set(BUILD_SUPPORT_DATA_DIR "${CMAKE_SOURCE_DIR}/build-support/data")

# Everything else in this section will populate the following global variables.
set(DDB_COMPILE_OPTIONS
    "-std=c++11"
    "-Wall"
    "-fno-builtin-malloc" "-fno-builtin-calloc" "-fno-builtin-realloc" "-fno-builtin-free"
    "-maes"  # SSE2
)
set(DDB_COMPILE_DEFINITIONS
    "-DLOCKFREE_SYMBASE"
    "-DOPENBLAS"
    "-D_GLIBCXX_USE_CXX11_ABI=0"
    "-DRUN_GTEST"
)
set(DDB_INCLUDE_DIRECTORIES
    ${PROJECT_SOURCE_DIR}/Include
    ${PROJECT_SOURCE_DIR}/src
    /usr/include/openssl
)
set(DDB_LINK_LIBRARIES
    DolphinDB
    z
    ssl
    crypto
    ${CMAKE_DL_LIBS}
    unwind
    pthread
    rt
    gtest
)
set(DDB_LINK_OPTIONS
    ""
)

# If your openssl is not at /usr/include/openssl, please export environment variable `DDB_OPENSSL_ROOT_DIR`
if(DEFINED ENV{DDB_OPENSSL_ROOT_DIR})
    set(OPENSSL_ROOT_DIR $ENV{DDB_OPENSSL_ROOT_DIR})
    find_package(OpenSSL REQUIRED)
    include_directories(${OPENSSL_INCLUDE_DIR})
    link_directories(${OPENSSL_ROOT_DIR}/lib/)
endif()

# Add compilation flags to DDB_COMPILE_OPTIONS based on the current CMAKE_BUILD_TYPE.
string(TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE)
if ("${CMAKE_BUILD_TYPE}" STREQUAL "DEBUG")               # debug
    list(APPEND DDB_COMPILE_DEFINITIONS "-DTEST" "-DVERBOSE_LOGGING")
    list(APPEND DDB_COMPILE_OPTIONS "-ggdb" "-O0" "-fno-omit-frame-pointer" "-fno-optimize-sibling-calls")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "RELEASE")         # release
    list(APPEND DDB_COMPILE_DEFINITIONS "-DNDEBUG")
    list(APPEND DDB_COMPILE_OPTIONS "-O3")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "RELWITHDEBINFO")  # release with debug info
    list(APPEND DDB_COMPILE_DEFINITIONS "-DNDEBUG")
    list(APPEND DDB_COMPILE_OPTIONS "-ggdb" "-O2")
else ()
    message(FATAL_ERROR "Unknown build type: ${CMAKE_BUILD_TYPE}")
endif ()
message(STATUS "CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}")

# ASAN, which includes LSAN.
if (${DDB_USE_ASAN})
    set(DDB_USE_TCMALLOC OFF)  # if enable asan, use system malloc

    set(DDB_ASAN_FLAGS
        "-fsanitize=address"                # Enable ASAN.
        "-fno-omit-frame-pointer"           # Nicer stack traces in error messages.
        "-fno-optimize-sibling-calls"       # Disable tail call elimination (perfect stack traces if inlining off).
        )
    list(APPEND DDB_COMPILE_OPTIONS ${DDB_ASAN_FLAGS})
    list(APPEND DDB_LINK_OPTIONS "-fsanitize=address")
    unset(DDB_ASAN_FLAGS)
endif ()
message(STATUS "ASAN: ${DDB_USE_ASAN}")
unset(DDB_ASAN_MSG)

# tcmalloc.
if (${DDB_USE_TCMALLOC})
    list(APPEND DDB_COMPILE_DEFINITIONS "-DTCMALLOC")
    list(APPEND DDB_LINK_LIBRARIES tcmalloc_minimal)  # Add to DolphinDB link libs.
else()
    list(APPEND DDB_COMPILE_DEFINITIONS "-DOSMALLOC")
endif ()
message(STATUS "tcmalloc: ${DDB_USE_TCMALLOC}")

# JIT.
if (${DDB_USE_JIT})
    find_package(LLVM 7 REQUIRED CONFIG)
    message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")
    message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")
    include_directories(${LLVM_INCLUDE_DIRS})

    list(APPEND DDB_COMPILE_DEFINITIONS "-DDOLPHINDB_JIT")
    list(APPEND DDB_LINK_LIBRARIES LLVM-7.1)
endif ()
message(STATUS "DDB_USE_JIT: ${DDB_USE_JIT}")

# platform.
if (APPLE)
    message(FATAL_ERROR "Unsupported platform: APPLE.")
elseif (UNIX)
    list(APPEND DDB_COMPILE_DEFINITIONS "-DLINUX")
elseif (WIN32)
    list(APPEND DDB_COMPILE_DEFINITIONS "-DWINDOWS")
else ()
    message(FATAL_ERROR "Unsupported platform.")
endif ()


#######################################################################################################################
# Dependencies: google test
#######################################################################################################################

set(OLD_CMAKE_BUILD_TYPE "${CMAKE_BUILD_TYPE}")     # Save the current CMAKE_BUILD_TYPE.
set(OLD_CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")           # Save the current CMAKE_C_FLAGS.
set(OLD_CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")       # Save the current CMAKE_CXX_FLAGS.

string(REPLACE ";" " " TEMP_COMPILE_FLAGS "${DDB_COMPILE_OPTIONS}")
set(CMAKE_C_FLAGS "${TEMP_COMPILE_FLAGS}")
set(CMAKE_CXX_FLAGS "${TEMP_COMPILE_FLAGS} -D_GLIBCXX_USE_CXX11_ABI=0")
unset(TEMP_COMPILE_FLAGS)

add_subdirectory(${PROJECT_SOURCE_DIR}/third_party/googletest-release-1.8.0)

# DISGUSTING HACK: Restore the old CMAKE_BUILD_TYPE, CMAKE_C_FLAGS, and CMAKE_CXX_FLAGS.
set(CMAKE_BUILD_TYPE "${OLD_CMAKE_BUILD_TYPE}")     # Restore the old CMAKE_BUILD_TYPE.
set(CMAKE_C_FLAGS "${OLD_CMAKE_C_FLAGS}")           # Restore the old CMAKE_C_FLAGS.
set(CMAKE_CXX_FLAGS "${OLD_CMAKE_CXX_FLAGS}")       # Restore the old CMAKE_CXX_FLAGS.
unset(OLD_CMAKE_C_FLAGS)                            # Variable hygiene.
unset(OLD_CMAKE_CXX_FLAGS)                          # Variable hygiene.
unset(OLD_CMAKE_BUILD_TYPE)                         # Variable hygiene.

# Add gtest to include path
list(APPEND DDB_INCLUDE_DIRECTORIES ${gtest_SOURCE_DIR}/include ${gtest_SOURCE_DIR})


#######################################################################################################################
# Build dolphindb executable
#######################################################################################################################

# Get the list of all DolphinDB sources.
file(GLOB_RECURSE
    DDB_SRCS                  # Store the list of files into the variable ${DDB_SRCS}.
    CONFIGURE_DEPENDS         # Ask CMake to regenerate the build system if these files change.
    ${PROJECT_SOURCE_DIR}/src/*.cpp
    ${PROJECT_SOURCE_DIR}/src/*.h
)
# Remove the JIT from DolphinDB sources.
if (NOT ${DDB_USE_JIT})
    list(REMOVE_ITEM DDB_SRCS ${PROJECT_SOURCE_DIR}/src/JIT/Codegen.cpp)
    list(REMOVE_ITEM DDB_SRCS ${PROJECT_SOURCE_DIR}/src/JIT/Codegen.h)
endif ()

# Get the list of all unit tests.
file(GLOB_RECURSE
    UNITTEST_SRCS                  # Store the list of files into the variable ${DDB_SRCS}.
    CONFIGURE_DEPENDS         # Ask CMake to regenerate the build system if these files change.
    ${PROJECT_SOURCE_DIR}/unittest2/*.cpp
    ${PROJECT_SOURCE_DIR}/unittest2/*.h
)

# Make sure that your call to link_directories takes place before your call to the relevant add_executable.
# ref: https://stackoverflow.com/questions/31438916/cmake-cannot-find-library-using-link-directories
link_directories(
    $ENV{DDB_LIB_DIR}
)

add_executable(dolphindb ${DDB_SRCS} ${UNITTEST_SRCS})

set_target_properties(dolphindb PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"       # Output the binaries to this folder.
)

target_compile_definitions(dolphindb PRIVATE ${DDB_COMPILE_DEFINITIONS})
target_compile_options(dolphindb PRIVATE ${DDB_COMPILE_OPTIONS})
target_include_directories(dolphindb PRIVATE ${DDB_INCLUDE_DIRECTORIES})
#
# `target_link_options` command was introduced in 3.13. version of cmake.
# If your cmake version is lower you may try using target_link_libraries instead.
# ref:
# - https://stackoverflow.com/questions/44320465/whats-the-proper-way-to-enable-addresssanitizer-in-cmake-that-works-in-xcode
# - https://github.com/DaisukeAtaraxiA/VSslnToCMake/issues/15
#
# target_link_options(dolphindb PRIVATE ${DDB_LINK_OPTIONS})
target_link_libraries(dolphindb ${DDB_LINK_OPTIONS} ${DDB_LINK_LIBRARIES})

if (DEFINED ENV{DDB_HOME})
    add_custom_command(TARGET dolphindb
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy "$ENV{DDB_HOME}/dolphindb.lic" "${CMAKE_BINARY_DIR}/bin/"
        COMMAND ${CMAKE_COMMAND} -E copy "$ENV{DDB_HOME}/dolphindb.dos" "${CMAKE_BINARY_DIR}/bin/"
    )
    if (EXISTS "$ENV{DDB_HOME}/dolphindb.cfg")
        add_custom_command(TARGET dolphindb
            POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy "$ENV{DDB_HOME}/dolphindb.cfg" "${CMAKE_BINARY_DIR}/bin/"
        )
    endif ()
endif ()


#######################################################################################################################
# Build test
#######################################################################################################################

include(CTest)          # CTest support is built into CMake.
enable_testing()        # CTest support is built into CMake.


function(add_dolphindb_test
        TEST_NAME                   # The name of this test.
        TEST_SOURCES                # The CPP files for this test.
        TEST_LABEL                  # The label of this test. Will be added as a dependency of this label.
        SHOULD_EXCLUDE_FROM_ALL     # EXCLUDE_ALL if we should exclude from default ALL target, NO_EXCLUDE otherwise.
        DEPENDS_ON_DDB              # Whether this test depends on DolphinDB runtime.
        )
    set(TEST_OUTPUT_DIR "${CMAKE_BINARY_DIR}/test")             # Output directory for tests.

    if (${SHOULD_EXCLUDE_FROM_ALL} STREQUAL "EXCLUDE_ALL")
        set(EXCLUDE_OPTION "EXCLUDE_FROM_ALL")
    elseif (${SHOULD_EXCLUDE_FROM_ALL} STREQUAL "NO_EXCLUDE")
        set(EXCLUDE_OPTION "")
    else ()
        message(FATAL_ERROR "Invalid option for SHOULD_EXCLUDE_FROM_ALL.")
    endif ()

    string(TOUPPER ${DEPENDS_ON_DDB} DEPENDS_ON_DDB)
    if (${DEPENDS_ON_DDB} STREQUAL "YES")
        list(APPEND TEST_SOURCES ${DDB_SRCS})
    endif ()

    add_executable(${TEST_NAME} ${EXCLUDE_OPTION} ${TEST_SOURCES})

    set_target_properties(${TEST_NAME} PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY "${TEST_OUTPUT_DIR}"       # Output the test binaries to this folder.
    )

    target_compile_options(${TEST_NAME} PRIVATE ${DDB_COMPILE_OPTIONS})
    target_compile_definitions(${TEST_NAME} PRIVATE ${DDB_COMPILE_DEFINITIONS} "-DRUN_GTEST")

    # Include the testing directories.
    target_include_directories(${TEST_NAME} PRIVATE ${gtest_SOURCE_DIR}/include ${gtest_SOURCE_DIR})
    target_link_libraries(${TEST_NAME} gtest)

    if (${DEPENDS_ON_DDB} STREQUAL "YES")
        target_include_directories(${TEST_NAME} PRIVATE ${DDB_INCLUDE_DIRECTORIES})
        target_link_libraries(${TEST_NAME} ${DDB_LINK_OPTIONS} ${DDB_LINK_LIBRARIES})
    else ()
        target_link_libraries(${TEST_NAME} ${CMAKE_DL_LIBS} pthread gtest_main)
    endif ()

    if ((${DEPENDS_ON_DDB} STREQUAL "YES") AND (DEFINED ENV{DDB_HOME}))
        add_custom_command(TARGET ${TEST_NAME}
            POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy "$ENV{DDB_HOME}/dolphindb.lic" "${TEST_OUTPUT_DIR}"
            COMMAND ${CMAKE_COMMAND} -E copy "$ENV{DDB_HOME}/dolphindb.dos" "${TEST_OUTPUT_DIR}"
        )
        if (EXISTS "$ENV{DDB_HOME}/dolphindb.cfg")
            add_custom_command(TARGET ${TEST_NAME}
                POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E copy "$ENV{DDB_HOME}/dolphindb.cfg" "${TEST_OUTPUT_DIR}"
            )
        endif ()
    endif ()

    add_test(NAME ${TEST_NAME} COMMAND ${TEST_NAME})
    # Label each test with TEST_LABEL so that ctest can run all the tests under the TEST_LABEL label later.
    set_tests_properties(${TEST_NAME} PROPERTIES
        LABELS "${TEST_LABEL};${TEST_NAME}"                 # Label the test.
    )
    # Add TEST_NAME as a dependency to TEST_LABEL. Note that TEST_LABEL must be a valid target!
    # add_dependencies(${TEST_LABEL} ${TEST_NAME})
endfunction()

add_dolphindb_test(mifirstnot_test ${PROJECT_SOURCE_DIR}/unittest/ComputingTest/MovingiFirstNot.cpp unittest EXCLUDE_ALL YES)
add_dolphindb_test(tsdb_test ${PROJECT_SOURCE_DIR}/unittest/TSDBTest/TSDBTest.cpp unittest EXCLUDE_ALL YES)


add_dolphindb_test(arithmetic_overflow_test ${PROJECT_SOURCE_DIR}/unittest/InfraTest/arithmetic_overflow_test.cpp unittest EXCLUDE_ALL NO)
add_dolphindb_test(decimal_test ${PROJECT_SOURCE_DIR}/unittest/InfraTest/decimal_test.cpp unittest EXCLUDE_ALL YES)

#######################################################################################################################
# format            :   Reformat the codebase according to standards.
# check-format      :   Check if the codebase is formatted according to standards.
#######################################################################################################################

find_program(CLANG_FORMAT_BIN NAMES clang-format-8 clang-format)
if ("${CLANG_FORMAT_BIN}" STREQUAL "CLANG_FORMAT_BIN-NOTFOUND")
    message(STATUS "[MISSING] clang-format not found, no format and no check-format.")
else ()
    # The directories to be formatted. Note that we modified the format script to take in multiple arguments.
    string(CONCAT FORMAT_DIRS
        "${PROJECT_SOURCE_DIR}/src,${PROJECT_SOURCE_DIR}/unittest"
    )

    # Run clang-format and update files in place.
    add_custom_target(format
            ${BUILD_SUPPORT_DIR}/run_clang_format.py
            ${CLANG_FORMAT_BIN}
            ${BUILD_SUPPORT_DATA_DIR}/clangformat_suppressions.txt
            --source_dirs
            ${FORMAT_DIRS}
            --fix
            --quiet
            USES_TERMINAL
            )

    # Run clang-format and exit with a non-zero exit code if any files need to be reformatted.
    add_custom_target(check-format
            ${BUILD_SUPPORT_DIR}/run_clang_format.py
            ${CLANG_FORMAT_BIN}
            ${BUILD_SUPPORT_DATA_DIR}/clangformat_suppressions.txt
            --source_dirs
            ${FORMAT_DIRS}
            --quiet
            USES_TERMINAL
            )

    message(STATUS "[ADDED] clang-format and check-clang-format (${CLANG_FORMAT_BIN})")

    unset(FORMAT_DIRS)
endif ()
unset(CLANG_FORMAT_BIN)

```

