cmake_minimum_required (VERSION 3.16)
project(agc)
include(ExternalProject)

## Require C++17 support
set (CMAKE_CXX_STANDARD 17)
set (CMAKE_CXX_STANDARD_REQUIRED ON)

## Output paths
set(LIBRARY_OUTPUT_PATH ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

## Determine build type and set flags accordingly
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release) ## Build Release by default
  set(c_cxx_shared_flags "-ffunction-sections -fdata-sections -Wl,-gc-sections -s -Wl,--strip-all -fmerge-all-constants -Wl,-z,norelro -fPIE -DNDEBUG -fno-exceptions")
endif()
set(c_cxx_shared_flags "${c_cxx_shared_flags} -fPIC -Wall -g -O3 -m64 -mavx")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${c_cxx_shared_flags}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${c_cxx_shared_flags} -fpermissive")

if (CMAKE_WITH_NATIVE_INSTRUCTIONS)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=native -mtune=native")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native -mtune=native")
elseif(NOT DEFINED CMAKE_WITH_NATIVE_INSTRUCTIONS)
  set(CMAKE_WITH_NATIVE_INSTRUCTIONS 0)
endif()

if(CMAKE_PGO_GENERATE AND NOT CMAKE_PGO_USE)
  if (CMAKE_C_COMPILER_ID STREQUAL "Clang")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-instr-generate -fcoverage-mapping")
  else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-generate")
  endif()
  if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-instr-generate -fcoverage-mapping")
  else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-generate")
  endif()
endif()

if(CMAKE_PGO_USE)
  if (CMAKE_C_COMPILER_ID STREQUAL "Clang")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-instr-use=${CMAKE_CURRENT_BINARY_DIR}/default.profdata -fprofile-correction")
  else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-use -fprofile-correction")
  endif()
  if (CMAKE_C_COMPILER_ID STREQUAL "Clang")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-instr-use=${CMAKE_CURRENT_BINARY_DIR}/default.profdata -fprofile-correction")
  else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-use -fprofile-correction")
  endif()
endif()

## Link time optimization
if(CMAKE_WITH_FLTO)
  cmake_policy(SET CMP0069 NEW)
  set(CMAKE_POLICY_DEFAULT_CMP0069 NEW)
  set(CMAKE_INTERPROCEDURAL_OPTIMIZATION 1)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -flto")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -flto")
endif()

cmake_policy(SET CMP0097 NEW)
set(CMAKE_POLICY_DEFAULT_CMP0097 NEW)

## libagc
add_library(libagc ${CMAKE_CURRENT_SOURCE_DIR}/src/lib-cxx/lib-cxx.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/core/agc_basic.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/core/agc_decompressor_lib.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/core/archive.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/core/collection.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/core/collection_v1.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/core/collection_v2.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/core/collection_v3.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/core/genome_io.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/core/lz_diff.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/core/segment.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/core/utils.cpp)
set(CMAKE_AGC_LIBRARY ${CMAKE_BINARY_DIR}/lib/libagc.a)
set_target_properties(libagc PROPERTIES OUTPUT_NAME agc)

add_executable(agc ${CMAKE_CURRENT_SOURCE_DIR}/src/app/main.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/app/application.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/core/agc_compressor.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/core/agc_decompressor.cpp)

target_link_libraries(agc libagc)

## Include project headers
if (NOT DEFINED CMAKE_AGC_HEADERS)
  set(CMAKE_AGC_HEADERS "${CMAKE_CURRENT_SOURCE_DIR}/src/app ${CMAKE_CURRENT_SOURCE_DIR}/src/core ${CMAKE_CURRENT_SOURCE_DIR}/src/lib-cxx")
endif()
include_directories(${CMAKE_AGC_HEADERS})

# ## Dependencies

## Threads
find_package(Threads REQUIRED)
target_link_libraries(libagc Threads::Threads)
target_link_libraries(agc Threads::Threads)

### zlib-ng
if (DEFINED CMAKE_ZLIB_HEADERS AND DEFINED CMAKE_ZLIB_LIBRARY)
  message(STATUS "zlib headers provided in: ${CMAKE_ZLIB_HEADERS}")
  message(STATUS "zlib library provided in: ${CMAKE_ZLIB_LIBRARY}")
elseif(ZLIB_FOUND)
  include_directories(${ZLIB_INCLUDE_DIRS})
  target_link_libraries(agc ${ZLIB_LIBRARIES})
  target_link_libraries(libagc ${ZLIB_LIBRARIES})
elseif(CMAKE_BUILD_ZLIB)
  ExternalProject_Add(zlib-ng
    GIT_REPOSITORY    https://github.com/zlib-ng/zlib-ng
    GIT_TAG           2.1.5
    PREFIX            "external"
    BINARY_DIR        "external/zlib-ng"
    BUILD_IN_SOURCE   0
    CMAKE_ARGS	     -D ZLIB_COMPAT=ON
  		     -D ZLIB_ENABLE_TESTS=OFF
		     -D ZLIBNG_ENABLE_TESTS=OFF
		     -D WITH_GTEST=OFF
		     -D BUILD_SHARED_LIBS=0
		     -D WITH_NATIVE_INSTRUCTIONS=${CMAKE_WITH_NATIVE_INSTRUCTIONS}
		     -D WITH_OPTIM=ON
		     -D WITH_NEW_STRATEGIES=ON
		     -D "CMAKE_C_FLAGS=${CMAKE_C_FLAGS}"
		     -D "CMAKE_C_COMPILER=${CMAKE_C_COMPILER}"
		     INSTALL_COMMAND   "")
  add_dependencies(libagc zlib-ng)
  add_dependencies(agc zlib-ng)
  set(CMAKE_ZLIB_HEADERS ${CMAKE_CURRENT_BINARY_DIR}/external/zlib-ng)
  set(CMAKE_ZLIB_LIBRARY ${CMAKE_CURRENT_BINARY_DIR}/external/zlib-ng/libz.a)
else()
  set(CMAKE_ZLIB_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/libs)
  if (UNIX AND NOT APPLE)
    set(CMAKE_ZLIB_LIBRARY ${CMAKE_CURRENT_SOURCE_DIR}/libs/cloudflare-zlib/libz.a)
  elseif(APPLE)
    set(CMAKE_ZLIB_LIBRARY ${CMAKE_CURRENT_SOURCE_DIR}/libs/cloudflare-zlib/libz.mac.a)
  endif()
endif()
include_directories(${CMAKE_ZLIB_HEADERS})
target_link_libraries(agc ${CMAKE_ZLIB_LIBRARY})
target_link_libraries(libagc ${CMAKE_ZLIB_LIBRARY})

### zstd
if (DEFINED CMAKE_ZSTD_HEADERS AND DEFINED CMAKE_ZSTD_LIBRARY)
  message(STATUS "zstd headers provided in: ${CMAKE_ZSTD_HEADERS}")
  message(STATUS "zstd library provided in: ${CMAKE_ZSTD_LIBRARY}")
elseif(ZSTD_FOUND)
  include_directories(${ZSTD_INCLUDE_DIRS})
  target_link_libraries(agc ${ZSTD_LIBRARIES})
  target_link_libraries(libagc ${ZSTD_LIBRARIES})
elseif(CMAKE_BUILD_ZSTD)
  ExternalProject_Add(zstd
    GIT_REPOSITORY    https://github.com/facebook/zstd
    GIT_TAG           v1.5.5
    PREFIX            "external"
    BUILD_IN_SOURCE   1
    CONFIGURE_COMMAND ""
    BUILD_COMMAND     CC=${CMAKE_C_COMPILER} CXX=${CMAKE_CXX_COMPILER} make CFLAGS=${CMAKE_C_FLAGS} CXXFLAGS=${CMAKE_CXX_FLAGS} ZSTD_STRIP_ERROR_STRINGS=1 ZSTD_LIB_DEPRECATED=0 ZSTD_NO_UNUSED_FUNCTIONS=1
    INSTALL_COMMAND   rm -rf ${CMAKE_CURRENT_BINARY_DIR}/external/zstd && cp -rf ${CMAKE_CURRENT_BINARY_DIR}/external/src/zstd/lib ${CMAKE_CURRENT_BINARY_DIR}/external/zstd)
  add_dependencies(libagc zstd)
  add_dependencies(agc zstd)
  set(CMAKE_ZSTD_HEADERS ${CMAKE_CURRENT_BINARY_DIR}/external/zstd)
  set(CMAKE_ZSTD_LIBRARY ${CMAKE_CURRENT_BINARY_DIR}/external/zstd/libzstd.a)
else()
  set(CMAKE_ZSTD_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/libs)
  if (UNIX AND NOT APPLE)
    set(CMAKE_ZSTD_LIBRARY ${CMAKE_CURRENT_SOURCE_DIR}/libs/libzstd.a)
  elseif(APPLE)
    set(CMAKE_ZSTD_LIBRARY ${CMAKE_CURRENT_SOURCE_DIR}/libs/libzstd.mac.a)
  endif()
endif()
include_directories(${CMAKE_ZSTD_HEADERS})
target_link_libraries(agc ${CMAKE_ZSTD_LIBRARY})
target_link_libraries(libagc ${CMAKE_ZSTD_LIBRARY})

## mimalloc
if (DEFINED CMAKE_MIMALLOC_HEADERS AND DEFINED CMAKE_MIMALLOC_LIBRARY)
  message(STATUS "mimalloc headers provided in: ${CMAKE_MIMALLOC_HEADERS}")
  message(STATUS "mimalloc library provided in: ${CMAKE_MIMALLOC_LIBRARY}")
elseif(MIMALLOC_FOUND)
  include_directories(${MIMALLOC_INCLUDE_DIRS})
  target_link_libraries(agc ${MIMALLOC_LIBRARIES})
  target_link_libraries(libagc ${MIMALLOC_LIBRARIES})
elseif(CMAKE_BUILD_MIMALLOC)
  ExternalProject_Add(mimalloc
    GIT_REPOSITORY    https://github.com/microsoft/mimalloc
    GIT_TAG           v2.1.2
    PREFIX            "external"
    BINARY_DIR        "external/mimalloc"
    BUILD_IN_SOURCE   0
    CMAKE_ARGS       -D MI_OVERRIDE=ON
                     -D MI_BUILD_SHARED=OFF
		     -D MI_BUILD_STATIC=ON
		     -D MI_BUILD_OBJECT=OFF
		     -D MI_BUILD_TESTS=OFF
		     -D "CMAKE_C_FLAGS=${CMAKE_C_FLAGS}"
		     -D "CMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}"
		     -D "CMAKE_C_COMPILER=${CMAKE_C_COMPILER}"
		     -D "CMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}"
    INSTALL_COMMAND   cp -rf ${CMAKE_CURRENT_BINARY_DIR}/external/src/mimalloc/include ${CMAKE_CURRENT_BINARY_DIR}/external/mimalloc/include)
  add_dependencies(libagc mimalloc)
  add_dependencies(agc mimalloc)
  set(CMAKE_MIMALLOC_HEADERS ${CMAKE_CURRENT_BINARY_DIR}/external/mimalloc/include)
  set(CMAKE_MIMALLOC_LIBRARY ${CMAKE_CURRENT_BINARY_DIR}/external/mimalloc/libmimalloc.a)
else()
  set(CMAKE_MIMALLOC_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/libs/libmimalloc)
  if (UNIX AND NOT APPLE)
    set(CMAKE_MIMALLOC_LIBRARY ${CMAKE_CURRENT_SOURCE_DIR}/libs/mimalloc/libmimalloc.a)
  elseif(APPLE)
    set(CMAKE_MIMALLOC_LIBRARY ${CMAKE_CURRENT_SOURCE_DIR}/libs/mimalloc/libmimalloc.mac.a)
  endif()
endif()
include_directories(${CMAKE_MIMALLOC_HEADERS})
target_link_libraries(agc ${CMAKE_MIMALLOC_LIBRARY})
target_link_libraries(libagc ${CMAKE_MIMALLOC_LIBRARY})

## Raduls
if (UNIX AND NOT APPLE)
  set(CMAKE_RADULS_LIBRARY ${CMAKE_CURRENT_SOURCE_DIR}/libs/libraduls.a)
elseif(APPLE)
  set(CMAKE_RADULS_LIBRARY ${CMAKE_CURRENT_SOURCE_DIR}/libs/libraduls.mac.a)
endif()
target_link_libraries(libagc ${CMAKE_RADULS_LIBRARY})
target_link_libraries(agc ${CMAKE_RADULS_LIBRARY})

# ## make install
install(TARGETS agc)
