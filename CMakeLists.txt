# Copyright (C) 2019 Yaoyuan <ibireme@gmail.com>.
# Released under the MIT License:
# https://github.com/ibireme/yyjson/blob/master/LICENSE

cmake_minimum_required(VERSION 3.5)
project(yyjson VERSION 0.3.0)


# ------------------------------------------------------------------------------
# Compile Options, see yyjson.h for more explanation.
option(YYJSON_DISABLE_READER "Disable JSON reader" OFF)
option(YYJSON_DISABLE_WRITER "Disable JSON writer" OFF)
option(YYJSON_DISABLE_FP_READER "Disable custom floating-point number reader" OFF)
option(YYJSON_DISABLE_FP_WRITER "Disable custom floating-point number writer" OFF)
option(YYJSON_DISABLE_COMMENT_READER "Disable non-standard comment reader" OFF)
option(YYJSON_DISABLE_INF_AND_NAN_READER "Disable non-standard inf and nan reader" OFF)


# ------------------------------------------------------------------------------
# Build Options
option(YYJSON_BUILD_TESTS "Build all tests" OFF)
option(YYJSON_BUILD_MISC "Build misc" OFF)
option(YYJSON_ENABLE_COVERAGE "Enable code coverage for tests" OFF)
option(YYJSON_ENABLE_VALGRIND "Enable valgrind memory checker for tests" OFF)
option(YYJSON_ENABLE_SANITIZE "Enable sanitizer for tests" OFF)


# ------------------------------------------------------------------------------
# Build Type
if(XCODE OR MSVC)
    set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING "" FORCE)
endif()
if(NOT CMAKE_BUILD_TYPE)
    message(STATUS "No build type selected, default to: Release")
    set(CMAKE_BUILD_TYPE Release)
endif()


# ------------------------------------------------------------------------------
# Library
add_library(yyjson src/yyjson.h src/yyjson.c)
target_include_directories(yyjson PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>)


# ------------------------------------------------------------------------------
# Compiler Flags
if(YYJSON_DISABLE_READER)
    add_definitions(-DYYJSON_DISABLE_READER=1)
endif()
if(YYJSON_DISABLE_WRITER)
    add_definitions(-DYYJSON_DISABLE_WRITER=1)
endif()
if(YYJSON_DISABLE_FP_READER)
    add_definitions(-DYYJSON_DISABLE_FP_READER=1)
endif()
if(YYJSON_DISABLE_FP_WRITER)
    add_definitions(-DYYJSON_DISABLE_FP_WRITER=1)
endif()
if(YYJSON_DISABLE_COMMENT_READER)
    add_definitions(-DYYJSON_DISABLE_COMMENT_READER=1)
endif()
if(YYJSON_DISABLE_INF_AND_NAN_READER)
    add_definitions(-DYYJSON_DISABLE_INF_AND_NAN_READER=1)
endif()


# ------------------------------------------------------------------------------
# Project Config
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
include(XcodeProperty)

if(XCODE)
    set_default_xcode_property(yyjson)
    set_xcode_deployment_version(yyjson "10.11" "9.0" "9.0" "2.0")

    set_xcode_property(yyjson GCC_C_LANGUAGE_STANDARD "c89")
    set_xcode_property(yyjson CLANG_CXX_LANGUAGE_STANDARD "c++98")
    
    set_xcode_property(yyjson OTHER_CFLAGS[variant=Debug] "-Wall -Wextra -Werror -pedantic -pedantic-errors")
    set_xcode_property(yyjson OTHER_CFLAGS[variant=Release] "-Wall -Wextra -Werror -pedantic -pedantic-errors")
elseif(MSVC)
    target_compile_options(yyjson PRIVATE $<$<C_COMPILER_ID:MSVC>:/permissive- /utf-8>)
    target_compile_options(yyjson PRIVATE $<$<CXX_COMPILER_ID:MSVC>:/permissive- /utf-8>)
endif()

if(BUILD_SHARED_LIBS)
    if(WIN32)
        target_compile_definitions(yyjson PUBLIC
            $<BUILD_INTERFACE:YYJSON_EXPORTS>
            $<INSTALL_INTERFACE:YYJSON_IMPORTS>)
    endif()
endif()


# ------------------------------------------------------------------------------
# Install
include(GNUInstallDirs)

install(TARGETS yyjson
        EXPORT yyjson-targets
        ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
        INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
install(EXPORT yyjson-targets
        FILE yyjson-config.cmake
        NAMESPACE yyjson::
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/yyjson")
install(FILES src/yyjson.h DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")


# ------------------------------------------------------------------------------
# Testing
if(YYJSON_BUILD_TESTS)
    enable_testing()
    
    if(XCODE)
        # Config XCTest
        find_package(XCTest REQUIRED)
        set(CMAKE_XCODE_ATTRIBUTE_CODE_SIGNING_ALLOWED "NO")
        set(CMAKE_XCODE_ATTRIBUTE_CODE_SIGNING_REQUIRED "NO")
        set(CMAKE_XCODE_ATTRIBUTE_CODE_SIGN_IDENTITY "")
        set(YYJSON_TEST_DATA ${CMAKE_CURRENT_SOURCE_DIR}/test/data)

        # Add test cases to XCTest
        file(GLOB YYJSON_TEST_SOURCE
            "test/test_*.c"
            "test/test_*.cpp"
        )
        set(YYJSON_TEST_LINES "")
        foreach(SRC_FILE ${YYJSON_TEST_SOURCE})
            string(REGEX REPLACE "(^.*/|\\.[^.]*$)" "" SRC_NAME ${SRC_FILE})
            set(YYJSON_TEST_LINES "${YYJSON_TEST_LINES}- (void)${SRC_NAME} {\n")
            set(YYJSON_TEST_LINES "${YYJSON_TEST_LINES}    extern void ${SRC_NAME}(void);\n")
            set(YYJSON_TEST_LINES "${YYJSON_TEST_LINES}    ${SRC_NAME}();\n")
            set(YYJSON_TEST_LINES "${YYJSON_TEST_LINES}}\n\n")
        endforeach()
        configure_file(
            "${CMAKE_CURRENT_SOURCE_DIR}/test/xctest/yy_xctest.m.in"
            "${CMAKE_CURRENT_SOURCE_DIR}/test/xctest/yy_xctest.m"
            @ONLY
        )
        unset(YYJSON_TEST_LINES)
        
        # Add source files and search path to XCTest
        file(GLOB YYJSON_TEST_SOURCE 
            "test/test_*.c"
            "test/test_*.cpp"
            "test/util/*.h"
            "test/util/*.c"
            "test/xctest/*"
        )
        xctest_add_bundle(yyjson_tests yyjson
            ${YYJSON_TEST_SOURCE}
            ${YYJSON_TEST_DATA}
        )
        set_target_properties(yyjson_tests PROPERTIES
            MACOSX_BUNDLE_INFO_PLIST ${CMAKE_CURRENT_SOURCE_DIR}/test/xctest/Info.plist
            RESOURCE ${YYJSON_TEST_DATA}
        )
        target_include_directories(yyjson_tests PRIVATE 
            ${CMAKE_CURRENT_SOURCE_DIR}/test/util
            ${CMAKE_CURRENT_SOURCE_DIR}/test/xctest
        )
        xctest_add_test(XCTest.yyjson yyjson_tests)
        
        set_default_xcode_property(yyjson_tests)
        set_xcode_deployment_version(yyjson_tests "10.11" "9.0" "9.0" "2.0")
        
    else()
        # Check valgrind command
        if (YYJSON_ENABLE_VALGRIND)
            find_program(MEMORYCHECK_COMMAND valgrind)
            if ("${MEMORYCHECK_COMMAND}" MATCHES "MEMORYCHECK_COMMAND-NOTFOUND")
                message(WARNING "Valgrind not found")
                unset(MEMORYCHECK_COMMAND)
            else()
                message(STATUS "Valgrind found")
                set(MEMORYCHECK_COMMAND_OPTIONS
                    --tool=memcheck
                    --leak-check=full
                    --trace-children=yes
                    --error-exitcode=1
                )
            endif()
        endif()
        
        # Copy test data
        file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/test/data" DESTINATION "${CMAKE_CURRENT_BINARY_DIR}")
        add_definitions(-DYYJSON_TEST_DATA_PATH="${CMAKE_CURRENT_BINARY_DIR}")
        
        # Add dependency
        add_library(yyjson_test_utils
            test/util/yy_test_utils.c
            test/util/david_gay_dtoa.c
        )
        target_include_directories(yyjson_test_utils PUBLIC test/util)
        
        # Add test cases
        file(GLOB YYJSON_TEST_SOURCE
            "test/test_*.c"
            "test/test_*.cpp"
        )
        foreach(SRC_FILE ${YYJSON_TEST_SOURCE})
            string(REGEX REPLACE "(^.*/|\\.[^.]*$)" "" SRC_NAME ${SRC_FILE})
            add_executable(${SRC_NAME} ${SRC_FILE})
            target_link_libraries(${SRC_NAME} PRIVATE yyjson yyjson_test_utils)

            if(MEMORYCHECK_COMMAND)
                add_test(NAME ${SRC_NAME}
                         COMMAND "${MEMORYCHECK_COMMAND}" ${MEMORYCHECK_COMMAND_OPTIONS} "${CMAKE_CURRENT_BINARY_DIR}/${SRC_NAME}")
            else()
                add_test(${SRC_NAME} ${SRC_NAME})
            endif()

            set_tests_properties(${SRC_NAME} PROPERTIES TIMEOUT 60)
            message(STATUS "Add test: ${SRC_NAME}")
        endforeach()
        
        # Add code coverage and sanitize
        if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
            if (YYJSON_ENABLE_COVERAGE)
                set(COMPILE_FLAGS --coverage)
                target_compile_options(yyjson PRIVATE ${COMPILE_FLAGS})
                target_link_libraries(yyjson INTERFACE ${COMPILE_FLAGS})
            endif()
            if (YYJSON_ENABLE_SANITIZE)
                set(COMPILE_FLAGS 
                    -fsanitize=address
                    -fsanitize=undefined
                    -fsanitize=leak
                    -fsanitize-recover=all
                    -fno-omit-frame-pointer
                    -fno-optimize-sibling-calls
                    -O1
                    -g
                )
                target_compile_options(yyjson PRIVATE ${COMPILE_FLAGS})
                target_link_libraries(yyjson INTERFACE ${COMPILE_FLAGS})
            endif()
        endif()
    
    endif()
    
    
    # Test compatibility
    if(MSVC)
        add_executable(test_compile_ansi_c test/compile_ansi.c)
        target_include_directories(test_compile_ansi_c PRIVATE src)
        add_executable(test_compile_ansi_cpp test/compile_ansi.cpp)
        target_include_directories(test_compile_ansi_cpp PRIVATE src)
        
        # set warning level 4, treat warnings as errors
        set(YYJSON_STRICT_FLAGS /W4 /WX)
        target_compile_options(test_compile_ansi_c PRIVATE 
            $<$<COMPILE_LANGUAGE:C>:${YYJSON_STRICT_FLAGS}>)
        target_compile_options(test_compile_ansi_cpp PRIVATE
            $<$<COMPILE_LANGUAGE:CXX>:${YYJSON_STRICT_FLAGS}>)
        
    elseif(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
        
        # Check ANSI C/C++ standard strictly, treat warnings as errors
        set(YYJSON_STRICT_C_FLAGS
            -pedantic
            -pedantic-errors
            -Werror
            -Wall
            -Wextra
            -Wmissing-prototypes
            -Wstrict-prototypes
            -Wdouble-promotion
            -Wconversion
            -Wundef
        )
        set(YYJSON_STRICT_CXX_FLAGS
            -pedantic
            -pedantic-errors
            -Werror
            -Wall
            -Wextra
            -Wdouble-promotion
            -Wconversion
            -Wundef
        )
        
        include(CheckCCompilerFlag)
        include(CheckCXXCompilerFlag)
        
        check_c_compiler_flag("-ansi" COMPILER_SUPPORTS_ANSI_C)
        if(COMPILER_SUPPORTS_ANSI_C)
            add_executable(test_compile_ansi_c test/compile_ansi.c)
            target_include_directories(test_compile_ansi_c PRIVATE src)
            target_compile_options(test_compile_ansi_c PRIVATE
                $<$<COMPILE_LANGUAGE:C>:-ansi ${YYJSON_STRICT_C_FLAGS}>)
        endif()
        
        check_cxx_compiler_flag("-ansi" COMPILER_SUPPORTS_ANSI_CPP)
        if(COMPILER_SUPPORTS_ANSI_CPP)
            add_executable(test_compile_ansi_cpp test/compile_ansi.cpp)
            target_include_directories(test_compile_ansi_cpp PRIVATE src)
            target_compile_options(test_compile_ansi_cpp PRIVATE
                $<$<COMPILE_LANGUAGE:CXX>:-ansi ${YYJSON_STRICT_CXX_FLAGS}>)
        endif()
        
        check_c_compiler_flag("-std=c11" COMPILER_SUPPORTS_C11)
        if(COMPILER_SUPPORTS_C11)
            add_executable(test_compile_c11 test/compile_ansi.c)
            target_include_directories(test_compile_c11 PRIVATE src)
            target_compile_options(test_compile_c11 PRIVATE
                $<$<COMPILE_LANGUAGE:C>:-std=c11 ${YYJSON_STRICT_C_FLAGS}>)
        endif()
        
        check_cxx_compiler_flag("-std=c++17" COMPILER_SUPPORTS_CPP17)
        if(COMPILER_SUPPORTS_CPP17)
            add_executable(test_compile_cpp17 test/compile_ansi.cpp)
            target_include_directories(test_compile_cpp17 PRIVATE src)
            target_compile_options(test_compile_cpp17 PRIVATE
                $<$<COMPILE_LANGUAGE:CXX>:-std=c++17 ${YYJSON_STRICT_CXX_FLAGS}>)
        endif()

    endif()

endif()


# ------------------------------------------------------------------------------
# Miscellaneous
if(YYJSON_BUILD_MISC)
    # jsoninfo
    add_executable(jsoninfo "misc/jsoninfo.c")
    target_link_libraries(jsoninfo PRIVATE yyjson)
    if(XCODE)
        set_default_xcode_property(jsoninfo)
    endif()

    # make tables
    find_package(GMP REQUIRED)
    find_package(MPFR REQUIRED)
    add_executable(make_tables "misc/make_tables.c")
    target_include_directories(make_tables PRIVATE 
        ${GMP_INCLUDE_DIR}
        ${MPFR_INCLUDES}
    )
    target_link_libraries(make_tables PRIVATE
        ${GMP_LIBRARIES}
        ${MPFR_LIBRARIES}
    )
    if(XCODE)
        set_default_xcode_property(make_tables)
    endif()
endif()
