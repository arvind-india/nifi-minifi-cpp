#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#

cmake_minimum_required(VERSION 2.6)

IF(POLICY CMP0048)
  CMAKE_POLICY(SET CMP0048 OLD)
ENDIF(POLICY CMP0048)

include_directories(../libminifi/include ../thirdparty/spdlog-20170710/include ../thirdparty/concurrentqueue ../thirdparty/yaml-cpp-yaml-cpp-20171024/include ../thirdparty/rapidjson-1.1.0/include ../thirdparty/)

include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if(COMPILER_SUPPORTS_CXX11)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Os")
elseif(COMPILER_SUPPORTS_CXX0X)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x -Os")
else()
 message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

add_executable(minifiexe MiNiFiMain.cpp)
if(THREADS_HAVE_PTHREAD_ARG)
  target_compile_options(PUBLIC minifiexe "-pthread")
endif()
if(CMAKE_THREAD_LIBS_INIT)
  target_link_libraries(minifiexe "${CMAKE_THREAD_LIBS_INIT}")
endif()

# Include UUID
find_package(UUID REQUIRED)

# Include OpenSSL
find_package(OpenSSL REQUIRED)
include_directories(${OPENSSL_INCLUDE_DIR})

# Link against minifi, yaml-cpp, civetweb-cpp, uuid, openssl, and rocksdb
#target_link_libraries(minifiexe core-minifi)

if (APPLE)
	target_link_libraries (minifiexe -Wl,-all_load core-minifi)
else ()
	target_link_libraries (minifiexe -Wl,--whole-archive core-minifi -Wl,--no-whole-archive)
endif ()

if (APPLE)
	target_link_libraries (minifiexe -Wl,-all_load minifi)
else ()
	target_link_libraries (minifiexe -Wl,--whole-archive minifi -Wl,--no-whole-archive)
endif ()

target_link_libraries(minifiexe yaml-cpp ${UUID_LIBRARIES} ${OPENSSL_LIBRARIES})

if (APPLE)
	get_property(extensions GLOBAL PROPERTY EXTENSION-OPTIONS)
	foreach(EXTENSION ${extensions})
		message("Linking MiNiFiMain against ${EXTENSION}")
		target_link_libraries (minifiexe -Wl,-all_load ${EXTENSION})
	endforeach()    
else ()
	get_property(extensions GLOBAL PROPERTY EXTENSION-OPTIONS)
	foreach(EXTENSION ${extensions})
	  target_link_libraries (minifiexe -Wl,--whole-archive ${EXTENSION} -Wl,--no-whole-archive)
	endforeach()
endif ()

set_target_properties(minifiexe
        PROPERTIES OUTPUT_NAME minifi)

install(TARGETS minifiexe
        RUNTIME
        DESTINATION bin
        COMPONENT bin)


add_custom_command(TARGET minifiexe POST_BUILD
           COMMAND cat ${CMAKE_BINARY_DIR}/all.log)
