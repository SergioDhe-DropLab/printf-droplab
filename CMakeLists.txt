cmake_minimum_required (VERSION 3.13)

include(CheckTypeSize)

project(
	printf
	LANGUAGES C
	DESCRIPTION "Self-contained C implementation of printf, vprintf, sprintf and related functions"
	HOMEPAGE_URL https://github.com/eyalroz/printf
	VERSION 6.2.0
)

option(BUILD_TESTS          "Build test programs for the library" ON)
option(BUILD_SHARED_LIBS    "Build the library as a shared (dynamically-linked) object rather than a static one " ON)

# Boolean options which go into config.h

option(SUPPORT_DECIMAL_SPECIFIERS             "Support decimal notation floating-point conversion specifiers (%f,%F)" ON)
option(SUPPORT_EXPONENTIAL_SPECIFIERS         "Support exponential floating point format conversion specifiers (%e,%E,%g,%G)" ON)
option(SUPPORT_MSVC_STYLE_INTEGER_SPECIFIERS  "Support the I + bit size integer specifiers (%I8, %I16, %I32, %I64) as in Microsoft Visual C++" ON)
option(SUPPORT_WRITEBACK_SPECIFIER            "Support the length write-back specifier (%n)" ON)
option(SUPPORT_LONG_LONG                      "Support long long integral types (allows for the ll length modifier and affects %p)" ON)
option(USE_DOUBLE_INTERNALLY                  "Use the C `double` type - typically 64-bit in size - for internal floating-point arithmetic " ON)
option(USE_FIXED_POINT                        "Use fixed point arithmetic for floating point operations" OFF)
option(CHECK_FOR_NUL_IN_FORMAT_SPECIFIER      "Be defensive in the undefined-behavior case of a format specifier not ending before the string ends" ON)

set(ALIASING_MODES NONE HARD SOFT)
set(ALIAS_STANDARD_FUNCTION_NAMES NONE CACHE STRING "Alias the standard library function names (printf, sprintf etc.) to the library's functions - concretely, via a macro, or not at all")
set_property(CACHE ALIAS_STANDARD_FUNCTION_NAMES PROPERTY STRINGS ${ALIASING_MODES})

#option(ALIAS_STANDARD_FUNCTION_NAMES          "Alias the standard library function names (printf, sprintf etc.) to the library's functions" ON)

if (NOT ${ALIAS_STANDARD_FUNCTION_NAMES} STREQUAL NONE)
	set("ALIAS_STANDARD_FUNCTION_NAMES_${ALIAS_STANDARD_FUNCTION_NAMES}" 1)
endif()

foreach(opt
	SUPPORT_DECIMAL_SPECIFIERS
	SUPPORT_EXPONENTIAL_SPECIFIERS
	SUPPORT_MSVC_STYLE_INTEGER_SPECIFIERS
	SUPPORT_WRITEBACK_SPECIFIER
	SUPPORT_LONG_LONG
	USE_DOUBLE_INTERNALLY
	USE_FIXED_POINT
	ALIAS_STANDARD_FUNCTION_NAMES_SOFT
	ALIAS_STANDARD_FUNCTION_NAMES_HARD
	CHECK_FOR_NUL_IN_FORMAT_SPECIFIER        
        )
	if (${${opt}})
		set("PRINTF_${opt}" 1)
	else()
		set("PRINTF_${opt}" 0)
	endif()
endforeach()

# Numeric defines which go into printf_config.h

set(PRINTF_INTEGER_BUFFER_SIZE     "32" CACHE STRING "Integer to string conversion buffer size")
set(PRINTF_DECIMAL_BUFFER_SIZE     "32" CACHE STRING "Floating-point to decimal conversion buffer size")
set(DEFAULT_FLOAT_PRECISION         "6" CACHE STRING "Default precision when printing floating-point values")
set(MAX_INTEGRAL_DIGITS_FOR_DECIMAL "9" CACHE STRING "Maximum number of integral-part digits of a floating-point value for which printing with %f uses decimal (non-exponential) notation")
set(LOG10_TAYLOR_TERMS              "4" CACHE STRING "The number of terms in a Taylor series expansion of log_10(x) to use for approximation")

# Checks related to the 'j', 'z' and 't' size modifiers

check_type_size( "long"        SIZEOF_LONG       )
check_type_size( "long long"   SIZEOF_LONG_LONG  )

set(ACCEPTABLE_JZT_TYPE_SIZES ${SIZEOF_LONG} ${SIZEOF_LONG_LONG})

function(validate_type_size type_name)
	check_type_size(${type_name} TYPE_SIZE)
	if (NOT ${TYPE_SIZE} IN_LIST ACCEPTABLE_JZT_TYPE_SIZES)
		message(FATAL_ERROR "sizeof(${type_name}) is  ${TYPE_SIZE}, which is neither sizeof(long) (${SIZEOF_LONG}) nor sizeof(long long) (${SIZEOF_LONG_LONG}). Please contact the library maintainers with your platform details.")
	endif()
endfunction()
validate_type_size("intmax_t")
validate_type_size("size_t")
validate_type_size("ptrdiff_t")

# Note we've defined BUILD_SHARED_LIBS, which affects the following
add_library(printf)

add_library("printf::printf" ALIAS printf)


set(GENERATED_INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/include")
configure_file("printf_config.h.in" "${GENERATED_INCLUDE_DIR}/printf_config.h" @ONLY)
target_sources(printf PRIVATE src/printf/printf.c "${GENERATED_INCLUDE_DIR}/printf_config.h" src/printf/printf.h)
target_compile_definitions(printf PRIVATE PRINTF_INCLUDE_CONFIG_H)
target_include_directories(printf PRIVATE "$<BUILD_INTERFACE:${GENERATED_INCLUDE_DIR}>")

set_property(TARGET printf PROPERTY C_STANDARD 99)
set_property(TARGET printf PROPERTY C_STANDARD_REQUIRED ON)
set_property(TARGET printf PROPERTY C_EXTENSIONS OFF)

target_include_directories(
	printf
	PRIVATE
	src
	PUBLIC
	"$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
	"$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/src/>"
)

set_target_properties(printf PROPERTIES
	LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib
	ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)

if (CMAKE_C_COMPILER_ID STREQUAL "MSVC")
	target_compile_options(printf PRIVATE /W4)
elseif (CMAKE_C_COMPILER_ID STREQUAL "GNU" OR
        CMAKE_C_COMPILER_ID STREQUAL "Clang")
	target_compile_options(printf PRIVATE -Wall -Wextra -pedantic -Wconversion)
	if (ALIAS_STANDARD_FUNCTION_NAMES)
		# This is important for preventing our aliased implementation
		# from being replaced, e.g. printf("%c", 'a') by putchar('a');
		# clang and GCC apparently do this as an optimization
		target_compile_options(printf PUBLIC -fno-builtin-printf)
	endif()
endif()

if (BUILD_TESTS)
	enable_testing()
	add_subdirectory(test)
endif()

if (UNIX)
	add_custom_target(printf-sizes
		COMMAND size -A -t $<TARGET_FILE:printf>  > printf_sizes.txt
		DEPENDS printf
		BYPRODUCTS printf_sizes.txt
		COMMENT Prints the sizes of the different sections of the ELF file: text, dat, vss etc.)

	add_custom_target(printf-symbols
		COMMAND nm --numeric-sort --print-size "$<TARGET_FILE:printf>" > printf_symbols.txt
		COMMAND bash -c "nm --numeric-sort --print-size $<TARGET_FILE:printf> | c++filt > printf_cpp_symbols.txt"
		VERBATIM
		DEPENDS printf
		BYPRODUCTS printf_symbols.txt printf_cpp_symbols.txt
		COMMENT Produces lists of the symbols, and C++demangled symbols, inside the library)

	add_custom_target(printf-lst
		COMMAND objdump --disassemble --line-numbers -S "$<TARGET_FILE:printf>" > printf.list
		DEPENDS printf
		BYPRODUCTS printf.lst
		COMMENT Dissassembles the compiled library into an .lst file)

endif()

# -------------------------
# Installation
# -------------------------

include(GNUInstallDirs)

# Note: No need for a config.cmake file for setting dependencies - as there
# are no dependencies; this library is self-contained

install(
	TARGETS printf
	EXPORT printf_export
	RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
	ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
	LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
	INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

install(
	FILES "src/printf/printf.h"
	DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/printf"
)

export(
	EXPORT printf_export
	NAMESPACE "printf::"
	FILE "${PROJECT_BINARY_DIR}/printf-targets.cmake"
)

install(
	EXPORT printf_export
	FILE "printf-targets.cmake"
	DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/printf"
	NAMESPACE "printf::"
)

include(CMakePackageConfigHelpers)

configure_package_config_file(
	"cmake/printf-config.cmake.in"
	"${CMAKE_CURRENT_BINARY_DIR}/printf-config.cmake"
	INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/printf"
)

write_basic_package_version_file(
	"printf-config-version.cmake"
	VERSION ${PROJECT_VERSION}
	COMPATIBILITY SameMinorVersion
)

install(
	FILES 
		"${CMAKE_CURRENT_BINARY_DIR}/printf-config-version.cmake"
		"${CMAKE_CURRENT_BINARY_DIR}/printf-config.cmake"
	DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/printf"
)
