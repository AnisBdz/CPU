
# This file specifies how the project should be built, using CMake.
# If you are unfamiliar with CMake, don't worry about all the details.
# The sections you might want to edit are marked as such, and
# the comments should hopefully make most of it clear.
#
# For many purposes, you may not need to change anything about this file.

cmake_minimum_required(VERSION 3.8)

# Set project name, version and laguages here. (change as needed)
# Version numbers are available by including "exampleConfig.h" in
# the source. See exampleConfig.h.in for some more details.
project(APP VERSION 0.0.1 LANGUAGES CXX)


# Include stuff. No change needed.
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/")
include(ConfigSafeGuards)
include(Colors)

# Include bison and flex
find_package(BISON)
find_package(FLEX)

# Include ftxui
include(FetchContent)

set(FETCHCONTENT_UPDATES_DISCONNECTED TRUE)
FetchContent_Declare(ftxui
  GIT_REPOSITORY https://github.com/ArthurSonzogni/ftxui
  GIT_TAG 7daeac25c003b14898f9ff9c6731ceb83ec55fd3
)

FetchContent_GetProperties(ftxui)
if(NOT ftxui_POPULATED)
  FetchContent_Populate(ftxui)
  add_subdirectory(${ftxui_SOURCE_DIR} ${ftxui_BINARY_DIR} EXCLUDE_FROM_ALL)
endif()


# --------------------------------------------------------------------------------
#                         Locate files (change as needed).
# --------------------------------------------------------------------------------
set(SOURCES
    src/core/helpers.cc
    src/core/register.cc
    src/core/operand.cc
    src/core/instruction.cc
    src/core/program.cc
    src/core/ram.cc
    src/core/alu.cc
    src/core/cu.cc
    src/core/cpu.cc
    src/core/fpu.cc
    src/core/vector_register.cc
    src/core/sse.cc
    src/core/loader.cc
)

set(TESTFILES
    tests/main.cc
    tests/helpers.test.cc
    tests/register.test.cc
    tests/program.test.cc
    tests/ram.test.cc
    tests/alu.test.cc
    tests/cpu.test.cc
    tests/fpu.test.cc
    tests/vectorial.test.cc
    tests/loader.test.cc
    tests/assembler.test.cc
    tests/examples.test.cc
)

#---------------------------------------------------------------------------------
#                                       UI
#---------------------------------------------------------------------------------

add_library(ui
    src/ui/ui.cc
    src/ui/padding.cc
)

target_include_directories(ui
    PUBLIC ${ftxui_SOURCE_DIR}/include
    PUBLIC ${PROJECT_SOURCE_DIR}/src/ui/headers
    PUBLIC ${PROJECT_SOURCE_DIR}/src/assembler/headers
    PUBLIC ${PROJECT_SOURCE_DIR}/src/core/headers
    PUBLIC ${CMAKE_CURRENT_BINARY_DIR}
)

#---------------------------------------------------------------------------------
#                                  Bison and Flex
#---------------------------------------------------------------------------------
BISON_TARGET(parser src/assembler/parser.yy  ${CMAKE_CURRENT_BINARY_DIR}/parser.cpp)
FLEX_TARGET(scanner src/assembler/scanner.ll ${CMAKE_CURRENT_BINARY_DIR}/scanner.cpp)
ADD_FLEX_BISON_DEPENDENCY(scanner parser)
add_library(assembler
    src/assembler/assembler.cc
    src/assembler/scanner.ll
    src/assembler/parser.yy
    src/assembler/driver.cc
    src/assembler/logger.cc
    ${BISON_parser_OUTPUTS}
    ${FLEX_scanner_OUTPUTS}
)

target_include_directories(assembler
    PUBLIC ${PROJECT_SOURCE_DIR}/src/assembler/headers
    PUBLIC ${PROJECT_SOURCE_DIR}/src/core/headers
    PUBLIC ${CMAKE_CURRENT_BINARY_DIR}
)


# --------------------------------------------------------------------------------
#                            Build! (Change as needed)
# --------------------------------------------------------------------------------
# Compile all sources into a library. Called core here (change if you wish).
add_library(core ${SOURCES})

# Engine needs its header files, and users of the library must also see these (PUBLIC). (No change needed)
target_include_directories(core
    PUBLIC ${PROJECT_SOURCE_DIR}/src/core/headers
    PUBLIC ${PROJECT_SOURCE_DIR}/lib
)

# Add an executable for the file main.cpp, here called app.
# If you add more executables, copy these lines accordingly.
add_executable(app src/main.cc)   # Name of exec. and location of file.

target_link_libraries(assembler PRIVATE core)
target_link_libraries(ui PRIVATE assembler)
target_link_libraries(ui PRIVATE ftxui::screen)
target_link_libraries(ui PRIVATE ftxui::dom)

target_link_libraries(app PRIVATE core) # Link the executable to `core` (if it uses it).
target_link_libraries(app PRIVATE assembler)
target_link_libraries(app PRIVATE ui)

# Set the compile options you want, possibly depending on compiler (change as needed).
# Do similar for the executables if you wish to set options for them as well.
target_compile_options(core PRIVATE
    # clang/GCC warnings
    $<$<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:GNU>>:
    -Wall
    -Wextra # reasonable and standard
    -Wshadow # warn the user if a variable declaration shadows one from a
    # parent context
    -Wnon-virtual-dtor # warn the user if a class with virtual functions has a
    # non-virtual destructor. This helps catch hard to
    # track down memory errors
    -Wold-style-cast # warn for c-style casts
    -Wcast-align # warn for potential performance problem casts
    -Wunused # warn on anything being unused
    -Woverloaded-virtual # warn if you overload (not override) a virtual
    # function
    -Wpedantic # warn if non-standard C++ is used
    -Wconversion # warn on type conversions that may lose data
    -Wmisleading-indentation # warn if identation implies blocks where blocks
    # do not exist
    -Wduplicated-cond # warn if if / else chain has duplicated conditions
    -Wduplicated-branches # warn if if / else branches have duplicated code
    -Wlogical-op # warn about logical operations being used where bitwise were
    # probably wanted
    -Wnull-dereference # warn if a null dereference is detected
    -Wuseless-cast # warn if you perform a cast to the same type
    -Wdouble-promotion # warn if float is implicit promoted to double
    -Wformat=2 # warn on security issues around functions that format output
    # (ie printf)
    >
    # MSVC warnings
    $<$<CXX_COMPILER_ID:MSVC>:
    "/Wall" >
    )

# --------------------------------------------------------------------------------
#                         Make Tests (no change needed).
# --------------------------------------------------------------------------------\
set(TEST_MAIN unit_tests) # Default name for test executable (change if you wish).

add_library(Catch2 INTERFACE)
target_include_directories(Catch2 INTERFACE lib/)

add_executable(${TEST_MAIN} ${TESTFILES})
target_link_libraries(${TEST_MAIN} Catch2 core assembler pthread)
add_custom_target(catch
    COMMAND "${PROJECT_BINARY_DIR}/${TEST_MAIN}"
    DEPENDS ${TEST_MAIN})

include(Catch)
catch_discover_tests(${TEST_MAIN})

# Set the properties you require, e.g. what C++ standard to use (change as needed).
set_target_properties(core app ui assembler ${TEST_MAIN} PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED YES
    CXX_EXTENSIONS NO
)
