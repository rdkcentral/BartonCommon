# ------------------------------ tabstop = 4 ----------------------------------
#
# If not stated otherwise in this file or this component's LICENSE file the
# following copyright and licenses apply:
#
# Copyright 2024 Comcast Cable Communications Management, LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
# SPDX-License-Identifier: Apache-2.0
#
# ------------------------------ tabstop = 4 ----------------------------------

cmake_minimum_required(VERSION 3.16.5)
project(barton-common)

# Determine if we are in our expected git repository or if we are integrated into another
execute_process(
                COMMAND git ls-remote --get-url origin
                WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
                OUTPUT_VARIABLE BARTON_COMMON_ORIGIN_REMOTE
                OUTPUT_STRIP_TRAILING_WHITESPACE
                RESULT_VARIABLE GIT_LSREMOTE_RESULT
                )

# Don't assume host, people use host url swaps via gitconfig insteadOf
if (GIT_LSREMOTE_RESULT EQUAL 0 AND BARTON_COMMON_ORIGIN_REMOTE MATCHES ".*rdkcentral/BartonCommon.git$")
    # Determine versions and print
    execute_process(
                    COMMAND git describe --tags --match [0-9]*.[0-9]*.[0-9]*
                    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
                    OUTPUT_VARIABLE BARTON_COMMON_VERSION
                    OUTPUT_STRIP_TRAILING_WHITESPACE
                    RESULT_VARIABLE GIT_DESCRIBE_RESULT
                    )

    if (GIT_DESCRIBE_RESULT EQUAL 0)
        message(STATUS "BartonCommon Version: ${BARTON_COMMON_VERSION}")
        string(REGEX MATCH "^[0-9]+\\.[0-9]+\\.[0-9]+" BARTON_COMMON_SO_VERSION ${BARTON_COMMON_VERSION})
        message(STATUS "BartonCommon SO Version: ${BARTON_COMMON_SO_VERSION}")
        string(REGEX MATCHALL "[0-9]+" BARTON_COMMON_VERSION_LIST ${BARTON_COMMON_SO_VERSION})

        list(GET BARTON_COMMON_VERSION_LIST 0 BARTON_COMMON_MAJOR_VERSION)
        list(GET BARTON_COMMON_VERSION_LIST 1 BARTON_COMMON_MINOR_VERSION)
        list(GET BARTON_COMMON_VERSION_LIST 2 BARTON_COMMON_PATCH_VERSION)

        # This is what you may also call the platform version. The focue is API/ABI compatibility.
        # As such, the minor version rarely changes. This is analog to GLib/GTK/GIR namespace
        # version schemeing where a semver of 4.24.3 has api version 4.0
        set(BARTON_COMMON_API_VERSION "${BARTON_COMMON_MAJOR_VERSION}.0")
        message(STATUS "BartonCommon API Version: ${BARTON_COMMON_API_VERSION}")
    else()
        message(WARNING "Could not determine BartonCommon version from git tags. Perhaps this is a shallow clone? Proceeding without version info.")
    endif()
else()
    message(WARNING "BartonCommon is integrated into another repository. Proceeding without version info.")
endif()

include(CTest)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_STANDARD 99)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE TRUE)

add_compile_definitions(_GNU_SOURCE)

if ("${CMAKE_PREFIX_PATH}" STREQUAL "")
    # This is a hacky way to deal with dependencies that don't
    # publish a pkg-config or cmake package file (otbr, matter).
    # This assumes that one is compiling for development, where
    # we also own the build/installation of those dependencies.
    set(CMAKE_PREFIX_PATH ${CMAKE_INSTALL_PREFIX})
endif()

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/config/cmake/modules")
include("${PROJECT_SOURCE_DIR}/config/cmake/DependencyVersions.cmake")

message(STATUS "Targetting system - ${CMAKE_SYSTEM_NAME}")
message(STATUS "Using C toolchain - ${CMAKE_C_COMPILER}")
message(STATUS "Using C++ toolchain - ${CMAKE_CXX_COMPILER}")

# Add custom target for running tests if it doesn't already exist.
# This will allow us to easily invoke all tests, as well as wrap
# them with code coverage if enabled
if (NOT TARGET runTests)
    add_custom_target(runTests
        COMMAND ${CMAKE_CTEST_COMMAND} --output-on-failure
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    )
endif()

add_subdirectory(libs)

# ── CMake package configuration export ──────────────────────────────────────
include(CMakePackageConfigHelpers)

configure_package_config_file(
        "${PROJECT_SOURCE_DIR}/config/cmake/BartonCommonConfig.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/BartonCommonConfig.cmake"
        INSTALL_DESTINATION lib/cmake/BartonCommon
)

if (BARTON_COMMON_SO_VERSION)
    write_basic_package_version_file(
            "${CMAKE_CURRENT_BINARY_DIR}/BartonCommonConfigVersion.cmake"
            VERSION ${BARTON_COMMON_SO_VERSION}
            COMPATIBILITY SameMajorVersion
    )

    install(
            FILES "${CMAKE_CURRENT_BINARY_DIR}/BartonCommonConfigVersion.cmake"
            DESTINATION lib/cmake/BartonCommon
    )
endif()

install(
        EXPORT BartonCommonTargets
        DESTINATION lib/cmake/BartonCommon
        NAMESPACE BartonCommon::
)

install(
        FILES "${CMAKE_CURRENT_BINARY_DIR}/BartonCommonConfig.cmake"
        DESTINATION lib/cmake/BartonCommon
)
