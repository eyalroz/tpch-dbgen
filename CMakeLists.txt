cmake_minimum_required(VERSION 3.0...3.28)
cmake_policy(SET CMP0054 NEW) # Don't try to interpret "FOO" as a variable
include(CheckIncludeFiles)
include(CheckTypeSize)
include(CheckSymbolExists)
include(CheckFunctionExists)

project(tpch-dbgen # TPC-H benchmark data (and query) generator utilities
	VERSION 2.17.3 # taken from the TPC-H benchmark release version
	LANGUAGES C)

if (NOT CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE STREQUAL "")
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "" FORCE)
endif()
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

# Values of properties will be forwarded into the config.h
set(DATABASE "DB2" CACHE STRING "Command language dialect to target with the qgen query generator (\"DATABASE\"")
set_property(CACHE DATABASE PROPERTY STRINGS INFORMIX DB2 TDAT SQLSERVER SYBASE ORACLE VECTORWISE POSTGRES MYSQL)

set(WORKLOAD TPCH CACHE STRING "Choice of benchmark / query workload (\"WORKLOAD\")")
set_property(CACHE WORKLOAD PROPERTY STRINGS TPCH)

set(CSV_OUTPUT_FORMAT OFF CACHE BOOL "Use CSV output format")
set(EOL_HANDLING OFF CACHE BOOL "Skip separator after the last field in generated lines?")

if (CSV_OUTPUT_FORMAT AND NOT EOL_HANDLING)
	message(WARNING "When CSV is used for the output format, EOL_HANDLING is ignored and a separator is _never_ printed after the last field on the line (the equivalent of EOL_HANDLING being ON).")
endif()

check_type_size("long long"        SIZEOF_LONG_LONG)
check_type_size("long"             SIZEOF_LONG)
check_type_size("int"              SIZEOF_INT)
check_type_size("short"            SIZEOF_SHORT)

check_include_file(strings.h       HAVE_STRINGS_H)
check_include_file(inttypes.h      HAVE_INTTYPES_H)
check_include_file(stdint.h        HAVE_STDINT_H)
check_include_file(sys/bittypes.h  HAVE_SYS_BITTYPES_H)

check_function_exists(getopt       HAVE_GETOPT)

configure_file(src/config.h.in src/config.h @ONLY)
# The following is necessary since the generated config.h will be placed
# in the build directory ("binary" directory), not in the source directory
include_directories("${CMAKE_CURRENT_BINARY_DIR}/src")

add_executable(dbgen
	src/bcd2.c
	src/bm_utils.c
	src/build.c
	src/driver.c
	src/load_stub.c
	src/permute.c
	src/print.c
	src/rnd.c
	src/rng64.c
	src/speed_seed.c
	src/text.c
)

add_executable(qgen
	src/bm_utils.c
	src/build.c
	src/permute.c
	src/qgen.c
	src/rnd.c
	src/rng64.c
	src/speed_seed.c
	src/text.c
	src/varsub.c
)

set_target_properties(dbgen qgen
	PROPERTIES
	C_STANDARD 99
	C_STANDARD_REQUIRED ON
	C_EXTENSIONS OFF)

if ("${CMAKE_C_COMPILER_ID}" STREQUAL "Clang")
	set_property(TARGET dbgen qgen APPEND PROPERTY COMPILE_OPTIONS -Wall -Wextra -Wno-missing-field-initializers)
	set_property(TARGET dbgen qgen APPEND PROPERTY COMPILE_DEFINITIONS _POSIX_C_SOURCE=200809L)
elseif ("${CMAKE_C_COMPILER_ID}" STREQUAL "GNU")
	set_property(TARGET dbgen qgen APPEND PROPERTY COMPILE_OPTIONS -Wall -Wextra -Wno-missing-field-initializers -Wno-format-truncation)
	set_property(TARGET dbgen qgen APPEND PROPERTY COMPILE_DEFINITIONS _POSIX_C_SOURCE=200809L)
elseif ("${CMAKE_C_COMPILER_ID}" STREQUAL "MSVC")
	set_property(TARGET dbgen qgen APPEND PROPERTY COMPILE_OPTIONS "/W3")
	set_property(TARGET dbgen qgen APPEND PROPERTY COMPILE_DEFINITIONS _CRT_NONSTDC_NO_DEPRECATE _CRT_SECURE_NO_WARNINGS)
endif ()

