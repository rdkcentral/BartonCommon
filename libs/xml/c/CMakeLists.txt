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

#--------------------------------------------------------------------------------------
#
# CMake definition for the fcore 'xml library'
# Should not be utilized directly, but imported from our parent
# file via the 'add_subdirectory()' routine.
#
#--------------------------------------------------------------------------------------


# define source files (using wildcard)
file(GLOB SOURCES "src/*.c")

pkg_check_modules(XML2 REQUIRED IMPORTED_TARGET libxml-2.0>=${XML_MIN_VERSION})

add_library(xhXmlHelper STATIC ${SOURCES})
add_library(BartonCommon::xhXmlHelper ALIAS xhXmlHelper)
target_link_libraries(xhXmlHelper
        PUBLIC PkgConfig::XML2
        PUBLIC xhTypes xhUtil)

target_include_directories(xhXmlHelper PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/public>
        $<INSTALL_INTERFACE:include>)

install(TARGETS xhXmlHelper EXPORT BartonCommonTargets)
install(DIRECTORY public/ DESTINATION include)

#TODO: test target
