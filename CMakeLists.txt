#
# Copyright (c) 2015-2016, Luca Fulchir<luca@fulchir.it>, All rights reserved.
#
# This file is part of "libRaptorQ".
#
# libRaptorQ is free software: you can redistribute it and/or modify
# it under the terms of the GNU Lesser General Public License as
# published by the Free Software Foundation, either version 3
# of the License, or (at your option) any later version.
#
# libRaptorQ is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# and a copy of the GNU Lesser General Public License
# along with libRaptorQ.  If not, see <http://www.gnu.org/licenses/>.

cmake_minimum_required (VERSION 2.8.12)
project (libRaptorQ)
enable_language(CXX)
enable_language(C)

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake; ${CMAKE_MODULE_PATH})
set(RQ_VERSION 0.2-prealpha)
set(RQ_ABI 1)
message(STATUS "libRaptorQ version ${RQ_VERSION}")

# default values
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release")
endif()


include(GNUInstallDirs)

option(DYNAMIC_LIB "Build dynamic library" ON)
option(STATIC_LIB "Build static library" ON)
option(CLANG_STDLIB "Use clang's libc++" OFF)

# defaults: only enable LTO/PROFILING with known compielrs
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU")
   option(LTO "Link Time Optimization" ON)
   option(PROFILING "Profiling: speedup library" ON)
else()
   option(LTO "Link Time Optimization" OFF)
   option(PROFILING "Profiling: speedup library" OFF)
endif()

message(STATUS "Build selected: ${CMAKE_BUILD_TYPE}")

if(NOT (STATIC_LIB MATCHES "ON"))
    if (NOT (DYNAMIC_LIB MATCHES "ON"))
        message(FATAL_ERROR "Do you want to actually build the library?")
    endif()
endif()


# with LTO each compiler needs its own AR/NM/RANLIB?
if(LTO MATCHES "ON")
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU")
        if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
            find_program(RQ_AR "llvm-ar")
            find_program(RQ_NM "llvm-nm")
            find_program(RQ_RANLIB "llvm-ranlib")
        elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
            find_program(RQ_AR "gcc-ar")
            find_program(RQ_NM "gcc-nm")
            find_program(RQ_RANLIB "gcc-ranlib")
        endif()
        set(CMAKE_AR ${RQ_AR})
        set(CMAKE_NM ${RQ_NM})
        set(CMAKE_RANLIB ${RQ_RANLIB})
        mark_as_advanced(FORCE RQ_AR RQ_NM RQ_RANLIB)
	    message(STATUS "Link Time Optimization activated")
    else()
        message(FATAL_ERROR "Sorry, don't know how to do LTO with your compiler")
    endif()
else()
	message(STATUS "Link Time Optimization deactivated")
endif()

# Profiling is pretty compiler-specific....
if(PROFILING MATCHES "ON")
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU")
        message(STATUS "Profiling activated")
        if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
            message(WARNING "Clang's profile support breaks deterministic builds!")
        endif()
    else()
        message(FATAL_ERROR "Sorry, don't know how to profile with your compiler")
    endif()
else()
    message(STATUS "Profiling deactivated")
endif()

if(CLANG_STDLIB MATCHES "ON")
	if(NOT (CMAKE_CXX_COMPILER_ID MATCHES "Clang"))
		message(FATAL_ERROR "libc++ is only supported by clang")
	endif()
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
	set(STDLIB stdc++)
elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
	if(CLANG_STDLIB MATCHES "ON")
		set(RQ_STDLIB_FLAG -stdlib=libc++)
		set(STDLIB c++)
	else()
		set(STDLIB stdc++)
	endif()
endif()

find_package(Threads REQUIRED)
find_package(git)
find_package(eigen REQUIRED)
include_directories(SYSTEM ${EIGEN3_INCLUDE_DIR})
find_package(RQ_LZ4 REQUIRED)
include_directories(SYSTEM ${RQ_LZ4_INCLUDE_DIR})
#lz4 build if necessary
include(${CMAKE_CURRENT_SOURCE_DIR}/external/build_deps.cmake)
# our include tree
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src)

# LATEX documentation
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/doc)


if(IS_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/.git/ AND GIT_FOUND)
    execute_process(COMMAND ${GIT_EXECUTABLE} rev-parse --sq HEAD OUTPUT_VARIABLE RQ_SEED)
else()
    set(RQ_SEED RaptorQ_nongit_v${RQ_VERSION})
    message(WARNING "Not inside a git repository. Compiler seed is now constant.")
endif()
set(DETERMINISTIC -frandom-seed=${RQ_SEED})

if(CMAKE_SYSTEM_NAME MATCHES "Windows")
    add_definitions(-DRQ_WINDOWS)
else()
    add_definitions(-DRQ_UNIX)
    # used by ftok to get a key. Track both the file and the version
    add_definitions(-DRQ_SHMPATH="${CMAKE_INSTALL_PREFIX}/lib/libRaptorQ.so-${RQ_VERSION}")
endif()


SET(SOURCES src/RaptorQ/v1/Operation.cpp
    		src/RaptorQ/v1/Thread_Pool.cpp
    		src/RaptorQ/v1/Bitmask.cpp
    		src/RaptorQ/v1/Rand.cpp
    		src/RaptorQ/v1/Parameters.cpp
    		src/RaptorQ/v1/Graph.cpp
    		src/RaptorQ/v1/Shared_Computation/Decaying_LF.cpp
    		src/RaptorQ/v1/Precode_Matrix_Instantiation.cpp
    		src/RaptorQ/v1/RFC.cpp
    		src/RaptorQ/v1/API_Wrapper_C.cpp)

SET(HEADERS	src/RaptorQ/RFC6330_v1.hpp
			src/RaptorQ/RFC6330_v1.h
			src/RaptorQ/RaptorQ_v1.hpp
			src/RaptorQ/v1/RaptorQ.hpp
			src/RaptorQ/v1/RaptorQ_Iterators.hpp
            src/RaptorQ/v1/Interleaver.hpp
    		src/RaptorQ/v1/Thread_Pool.hpp
    		src/RaptorQ/v1/multiplication.hpp
    		src/RaptorQ/v1/table2.hpp
    		src/RaptorQ/v1/degree.hpp
    		src/RaptorQ/v1/common.hpp
    		src/RaptorQ/v1/Encoder.hpp
    		src/RaptorQ/v1/Decoder.hpp
    		src/RaptorQ/v1/Rand.hpp
    		src/RaptorQ/v1/Operation.hpp
    		src/RaptorQ/v1/Precode_Matrix.hpp
    		src/RaptorQ/v1/Precode_Matrix_Init.hpp
    		src/RaptorQ/v1/Precode_Matrix_Solver.hpp
    		src/RaptorQ/v1/Parameters.hpp
    		src/RaptorQ/v1/Graph.hpp
    		src/RaptorQ/v1/De_Interleaver.hpp
    		src/RaptorQ/v1/RFC.hpp
			src/RaptorQ/v1/API_Iterators.hpp
    		src/RaptorQ/v1/Shared_Computation/Decaying_LF.hpp
    		src/RaptorQ/v1/Shared_Computation/LZ4_Wrapper.hpp
			src/RaptorQ/v1/API_Wrapper_Raw.hpp
			src/RaptorQ/v1/API_Wrapper.hpp
    		src/RaptorQ/v1/API_Wrapper.h)


include(cmake/compiler_flags.cmake)

if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    add_definitions(-DUSING_CLANG)
    set(LD_OPT "-flto -fuse-ld=gold" )
elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    set(LD_OPT "")
endif()


########################################
## TODO: if windows, add notelemetry.obj
########################################


if(PROFILING MATCHES "ON")
    if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        set(PROFILE_GET           -fprofile-instr-generate)
        set(PROFILE_GET_FILE      RaptorQ.profraw)
        set(PROFILE_SET_FILE      RaptorQ.profdata)
        set(PROFILE_SET           "-fprofile-instr-use=${PROFILE_SET_FILE}")
    elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
        set(PROFILE_GET           -fprofile-generate)
        # gcc creates a couple more files than clang for prifiling. Track them.
        set(PROFILE_GET_FILE
CMakeFiles/test_c_profiled.dir/test/test_c.c.gcda
CMakeFiles/RaptorQ_Static_Profiling.dir/src/RaptorQ/v1/Shared_Computation/Decaying_LF.cpp.gcda
CMakeFiles/RaptorQ_Static_Profiling.dir/src/RaptorQ/v1/Graph.cpp.gcda
CMakeFiles/RaptorQ_Static_Profiling.dir/src/RaptorQ/v1/Rand.cpp.gcda
CMakeFiles/RaptorQ_Static_Profiling.dir/src/RaptorQ/v1/Operation.cpp.gcda
CMakeFiles/RaptorQ_Static_Profiling.dir/src/RaptorQ/v1/RFC.cpp.gcda
CMakeFiles/RaptorQ_Static_Profiling.dir/src/RaptorQ/v1/Precode_Matrix_Instantiation.cpp.gcda
CMakeFiles/RaptorQ_Static_Profiling.dir/src/RaptorQ/v1/Parameters.cpp.gcda
CMakeFiles/RaptorQ_Static_Profiling.dir/src/RaptorQ/v1/Bitmask.cpp.gcda
CMakeFiles/RaptorQ_Static_Profiling.dir/src/RaptorQ/v1/Thread_Pool.cpp.gcda
CMakeFiles/RaptorQ_Static_Profiling.dir/src/RaptorQ/v1/API_Wrapper_C.cpp.gcda)
        set(PROFILE_SET_FILE      ${PROFILE_GET_FILE})
        set(PROFILE_SET           -fprofile-use)
    endif()

    # PRE-run: build a library, generate the profile.
    add_library(RaptorQ_Static_Profiling STATIC ${SOURCES} ${HEADERS})
    target_link_libraries(RaptorQ_Static_Profiling ${STDLIB} ${RQ_LZ4_DEP} ${RQ_UBSAN})
	add_dependencies(RaptorQ_Static_Profiling LZ4)
    target_compile_options(
        RaptorQ_Static_Profiling PRIVATE
        ${CXX_COMPILER_FLAGS}
		${PROFILE_GET}
    )
    set_target_properties(RaptorQ_Static_Profiling PROPERTIES COMPILER_FLAGS ${PROFILE_GET})

    # build the C example test for profiling
    add_executable(test_c_profiled test/test_c.c)
    add_dependencies(test_c_profiled RaptorQ_Static_Profiling)
    target_compile_options(
        test_c_profiled PRIVATE
        ${C_COMPILER_FLAGS}
		${PROFILE_GET}
    )
    target_link_libraries(test_c_profiled RaptorQ_Static_Profiling ${STDLIB} m ${CMAKE_THREAD_LIBS_INIT})
    set_target_properties(test_c_profiled PROPERTIES LINK_FLAGS ${PROFILE_GET})

    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        add_custom_command(
            OUTPUT ${PROFILE_GET_FILE}
            COMMAND LLVM_PROFILE_FILE=${PROFILE_GET_FILE} ./test_c_profiled
            DEPENDS test_c_profiled
            COMMENT "Running profiling test..."
            VERBATIM
        )
    elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
        add_custom_command(
            OUTPUT ${PROFILE_GET_FILE}
            COMMAND ./test_c_profiled
            DEPENDS test_c_profiled
            COMMENT "Running profiling test..."
            VERBATIM
        )
    endif()
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        # the profile must be translated into readable form
        add_custom_command(
            OUTPUT ${PROFILE_SET_FILE}
            COMMAND llvm-profdata merge -output=${PROFILE_SET_FILE} ${PROFILE_GET_FILE}
            DEPENDS ${PROFILE_GET_FILE}
            COMMENT "Creating profile data..."
            VERBATIM
        )
    endif()
    add_custom_target(
        profile ALL
        DEPENDS ${PROFILE_SET_FILE}
    )
endif()



# build the static library
if(STATIC_LIB MATCHES "ON")
    add_library(RaptorQ_Static STATIC ${SOURCES} ${HEADERS})
	add_dependencies(RaptorQ_Static LZ4)
    if(NOT CMAKE_SYSTEM_NAME MATCHES "Windows")
        set_target_properties(RaptorQ_Static PROPERTIES OUTPUT_NAME RaptorQ.${RQ_ABI})
    endif()
    target_compile_options(
        RaptorQ_Static PRIVATE
        ${CXX_COMPILER_FLAGS}
    )
    if(LTO MATCHES "ON")
        target_compile_options(RaptorQ_Static PRIVATE -flto)
    endif()
    if(PROFILING MATCHES "ON")
        add_dependencies(RaptorQ_Static profile)
        target_compile_options(RaptorQ_Static PRIVATE ${PROFILE_SET})
    endif()
    set_target_properties (
        RaptorQ_Static
        PROPERTIES
        LINK_FLAGS "-Wl,-z,now,-z,relro -pie ${LD_OPT}"
    )
    set_property(TARGET RaptorQ_Static PROPERTY ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/lib")

	# make the library deterministic (only on *nix)
    add_executable(make_deterministic src/deterministic.cpp)
    target_compile_options(
        make_deterministic PRIVATE
        ${CXX_COMPILER_FLAGS}
    )
	target_link_libraries(make_deterministic ${STDLIB} ${RQ_UBSAN})
    add_custom_command(
        OUTPUT deterministic.run
        COMMAND make_deterministic ${CMAKE_CURRENT_BINARY_DIR}/lib/libRaptorQ.${RQ_ABI}.a
        DEPENDS RaptorQ_Static
        COMMENT "Removing creation date from library..."
        VERBATIM
    )

	add_custom_target(
	    Make_static_deterministic ALL
	    DEPENDS deterministic.run
	)
endif()

#build dynamic library
if(DYNAMIC_LIB MATCHES "ON")
    add_definitions(-DRAPTORQ_DLL)
    add_definitions(-DRAPTORQ_DLL_EXPORTS)

    add_library(RaptorQ SHARED ${SOURCES} ${HEADERS})
	add_dependencies(RaptorQ LZ4)
    target_compile_options(
        RaptorQ PRIVATE
        ${CXX_COMPILER_FLAGS}
    )
    if(LTO MATCHES "ON")
        target_compile_options(RaptorQ PRIVATE -flto)
    endif()
    if(PROFILING MATCHES "ON")
        add_dependencies(RaptorQ profile)
        target_compile_options(RaptorQ PRIVATE ${PROFILE_SET})
    endif()
    set_target_properties (
        RaptorQ
        PROPERTIES
        LINK_FLAGS "-Wl,-z,now,-z,relro ${LD_OPT}"
		SOVERSION ${RQ_ABI}
		VERSION ${RQ_VERSION}
	)
    set_property(TARGET RaptorQ PROPERTY LIBRARY_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/lib")
endif()


# add main tests
add_executable(libRaptorQ-test EXCLUDE_FROM_ALL test/rfc_test.cpp)
target_compile_options(
	libRaptorQ-test PRIVATE
	${CXX_COMPILER_FLAGS}
)
add_dependencies(libRaptorQ-test RaptorQ)
target_link_libraries(libRaptorQ-test ${RQ_UBSAN} ${STDLIB} RaptorQ ${CMAKE_THREAD_LIBS_INIT} ${RQ_LZ4_DEP})
add_custom_target(tests DEPENDS libRaptorQ-test)

# build examples
# C interface
add_executable(test_c EXCLUDE_FROM_ALL test/test_c.c)
target_compile_options(
	test_c PRIVATE
	${C_COMPILER_FLAGS}
	-Wno-disabled-macro-expansion
)
add_dependencies(test_c RaptorQ)
target_link_libraries(test_c RaptorQ m ${RQ_UBSAN} ${STDLIB} ${CMAKE_THREAD_LIBS_INIT} ${RQ_LZ4_DEP})

# CPP interface - RFC interface
add_executable(test_cpp_rfc EXCLUDE_FROM_ALL test/test_cpp_rfc.cpp)
target_compile_options(
	test_cpp_rfc PRIVATE
	${CXX_COMPILER_FLAGS}
)
add_dependencies(test_cpp_rfc RaptorQ)
target_link_libraries(test_cpp_rfc ${RQ_UBSAN} ${STDLIB} RaptorQ ${CMAKE_THREAD_LIBS_INIT} ${RQ_LZ4_DEP})

# CPP interface - RAW interface
add_executable(test_cpp_raw EXCLUDE_FROM_ALL test/test_cpp_raw.cpp)
target_compile_options(
	test_cpp_raw PRIVATE
	${CXX_COMPILER_FLAGS}
)
add_dependencies(test_cpp_raw RaptorQ)
target_link_libraries(test_cpp_raw ${RQ_UBSAN} ${STDLIB} RaptorQ ${CMAKE_THREAD_LIBS_INIT} ${RQ_LZ4_DEP})
add_custom_target(examples DEPENDS test_c test_cpp_rfc test_cpp_raw)

add_custom_target(everything DEPENDS Make_static_deterministic tests examples docs)


install(FILES ${HEADERS} DESTINATION include/RaptorQ/)
install(TARGETS RaptorQ RaptorQ_Static
     LIBRARY DESTINATION lib COMPONENT libraries
     ARCHIVE DESTINATION lib COMPONENT libraries)


