cmake_minimum_required(VERSION 3.3)
project(clunk LANGUAGES C CXX VERSION 1.3)

set(CMAKE_USE_RELATIVE_PATHS ON)
set(CMAKE_CXX_STANDARD 11)

include(GNUInstallDirs)

option(BUILD_TEST "Build simple test application" OFF)
option(WITH_SSE "Use highly optimized SSE FFT/MDCT routines" OFF)
option(BUILD_STATIC "Build static library" OFF)

set(WITH_SDL SDL2 CACHE STRING "Enable SDL/SDL2 backend")
set_property(CACHE WITH_SDL PROPERTY STRINGS SDL SDL2 OFF)

if (WITH_SDL STREQUAL "SDL2")
	message(STATUS "building with SDL2 support...")
	find_package(SDL2 REQUIRED)
	include_directories(${SDL2_INCLUDE_DIRS})
	set(LIBRARIES ${SDL2_LIBRARIES})
elseif (WITH_SDL STREQUAL "SDL")
	message(STATUS "building with SDL1 support...")
	find_package(SDL REQUIRED)
	include_directories(${SDL_INCLUDE_DIR})
	set(LIBRARIES ${SDL_LIBRARY})
endif()

message(STATUS "build type: ${CMAKE_BUILD_TYPE}")
if(${CMAKE_BUILD_TYPE} MATCHES Debug)
	add_definitions(-DDEBUG)
	add_definitions(-ggdb -Wall -pedantic -Wno-unused-variable)
endif (${CMAKE_BUILD_TYPE} MATCHES Debug)

if("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU" OR "${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
	set(CMAKE_CXX_VISIBILITY_PRESET hidden)
	set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)
	add_definitions("-DGCC_HASCLASSVISIBILITY=1")
endif()

if(WIN32)
	add_definitions(-D_CRT_SECURE_NO_WARNINGS -D_WINDOWS)
endif(WIN32)

add_definitions(-DCLUNKAPI=DLLEXPORT)
set(CMAKE_INCLUDE_CURRENT_DIR TRUE)

set(SOURCES
	clunk/buffer.cpp
	clunk/clunk_ex.cpp
	clunk/context.cpp
	clunk/distance_model.cpp
	clunk/hrtf.cpp
	clunk/kemar.c
	clunk/logger.cpp
	clunk/object.cpp
	clunk/sample.cpp
	clunk/source.cpp
	clunk/stream.cpp
	clunk/wav_file.cpp
#	clunk/clunk_c.cpp
)

set(PUBLIC_HEADERS
	clunk/buffer.h
	clunk/clunk.h
	clunk/clunk_assert.h
	clunk/context.h
	clunk/distance_model.h
	clunk/export_clunk.h
	clunk/fft_context.h
	clunk/hrtf.h
	clunk/kemar.h
	clunk/locker.h
	clunk/logger.h
	clunk/mdct_context.h
	clunk/object.h
	clunk/ref_mdct_context.h
	clunk/sample.h
	clunk/source.h
	clunk/sse_fft_context.h
	clunk/stream.h
	clunk/v3.h
	clunk/clunk_c.h
	clunk/window_function.h
	${CMAKE_CURRENT_BINARY_DIR}/clunk/config.h
)

if (SDL_FOUND OR SDL2_FOUND)
	set(CLUNK_BACKEND_SDL 1)
	list(APPEND SOURCES
		clunk/backend/sdl/sdl_ex.cpp
		clunk/backend/sdl/backend.cpp
	)
	list(APPEND PUBLIC_HEADERS
		clunk/backend/sdl/backend.h
	)
endif ()

if (WITH_SSE)
	list(APPEND SOURCES clunk/sse_fft_context.cpp)
	set(CLUNK_USES_SSE 1)
endif(WITH_SSE)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/clunk/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/clunk/config.h)

add_library(clunk SHARED
	${SOURCES}
)

target_include_directories(clunk PUBLIC ${CMAKE_CURRENT_BINARY_DIR})
target_include_directories(clunk PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_link_libraries(clunk ${LIBRARIES})

if(BUILD_STATIC)
	add_library(clunk-static STATIC
		${SOURCES}
	)

	target_include_directories(clunk-static PUBLIC ${CMAKE_CURRENT_BINARY_DIR})
	target_include_directories(clunk-static PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
endif()

install(TARGETS clunk)
install(FILES ${PUBLIC_HEADERS} DESTINATION include/clunk)

if(BUILD_TEST)
	if (WITH_SDL STREQUAL "OFF")
		message(FATAL_ERROR "you need SDL library to build test")
	endif ()

	add_executable(clunk-test test.cpp)
	target_link_libraries(clunk-test clunk)

	add_executable(clunk-mdct clunk-mdct.cpp)
	target_link_libraries(clunk-mdct clunk)
endif(BUILD_TEST)
