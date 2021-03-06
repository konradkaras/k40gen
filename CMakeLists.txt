cmake_minimum_required(VERSION 3.8 FATAL_ERROR)
project(k40gen C CXX)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake) # for find_package

option(ENABLE_PYTHON "Enable python bindings" TRUE)
option(ENABLE_VC "Enable usage of VC library" TRUE)
option(ENABLE_TESTS "Enable tests" TRUE)

include(ExternalProject)
include(FindPackageHandleStandardArgs)

set(EXTERNAL_DIR "external")

if (ENABLE_VC)
find_package(Vc QUIET)

if (NOT Vc_FOUND OR "${Vc_VERSION}" VERSION_LESS "1.3.2")

message(STATUS "Using builtin Vc")
set(Vc_BUILTIN TRUE)
set(Vc_VERSION "1.4.1")
set(Vc_PROJECT "Vc-${Vc_VERSION}")
set(Vc_SRC_URI "https://github.com/VcDevel/Vc/archive/${Vc_VERSION}.tar.gz")
set(Vc_DESTDIR "${CMAKE_BINARY_DIR}/${EXTERNAL_DIR}")
set(Vc_LIBNAME "${CMAKE_STATIC_LIBRARY_PREFIX}Vc${CMAKE_STATIC_LIBRARY_SUFFIX}")
set(Vc_LIBRARY "${Vc_DESTDIR}/lib/${Vc_LIBNAME}")


ExternalProject_Add(VC
  URL ${Vc_SRC_URI}
  URL_HASH SHA256=7e8b57ed5ff9eb0835636203898c21302733973ff8eaede5134dd7cb87f915f6
  INSTALL_DIR ${Vc_DESTDIR}
  BUILD_IN_SOURCE 0
  BUILD_BYPRODUCTS ${Vc_LIBRARY}
  LOG_DOWNLOAD 1 LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1
  CMAKE_ARGS -G ${CMAKE_GENERATOR}
             -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
             -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
             -DCMAKE_C_FLAGS=${CMAKE_C_FLAGS}
             -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
             -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
             -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
	     STEP_TARGETS install
)

set(VC_TARGET Vc)
set(Vc_LIBRARIES Vc)
set(Vc_INCLUDE_DIR "${Vc_DESTDIR}/include")
set(Vc_CMAKE_MODULES_DIR "${Vc_DESTDIR}/lib/cmake/Vc")

find_package_handle_standard_args(Vc
  FOUND_VAR Vc_FOUND
  REQUIRED_VARS Vc_INCLUDE_DIR Vc_LIBRARIES Vc_CMAKE_MODULES_DIR
  VERSION_VAR Vc_VERSION)

else()
  set(Vc_BUILTIN FALSE)
endif()
else()
  set(Vc_BUILTIN FALSE)
  set(USE_AVX2 FALSE)
endif()

# vectorclass
set(vectorclass_VERSION "1.30")
set(vectorclass_SRC_URI "https://www.agner.org/optimize/vectorclass.zip")
set(vectorclass_DESTDIR "${CMAKE_BINARY_DIR}/include")
set(vectorclass_ROOTDIR "${vectorclass_DESTDIR}/vectorclass")

ExternalProject_Add(vectorclass
  URL ${vectorclass_SRC_URI}
  URL_HASH SHA256=f9cb70a3e865dd019b58f449d11f90147ce8ba5f2c60410389ec0ead92944b97
  SOURCE_DIR vectorclass
  INSTALL_DIR ${vectorclass_ROOTDIR}
  LOG_DOWNLOAD 1 LOG_BUILD 1 LOG_INSTALL 1
  CONFIGURE_COMMAND ""
  BUILD_COMMAND unzip -d <SOURCE_DIR> -o <SOURCE_DIR>/special.zip
  INSTALL_COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR> <INSTALL_DIR>
  STEP_TARGETS install
)

# FIXME: This is a workaround to let ROOT find the headers at runtime if
# they are in the build directory. This is necessary until we decide how to
# treat externals with headers used by ROOT
if(NOT EXISTS ${CMAKE_BINARY_DIR}/include/Vc)
  if (NOT EXISTS ${CMAKE_BINARY_DIR}/include)
    execute_process(COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/include)
  endif()
  execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink
    ${Vc_INCLUDE_DIR}/Vc ${CMAKE_BINARY_DIR}/include/Vc)
endif()
# end of workaround

# install(DIRECTORY ${Vc_DESTDIR}/ DESTINATION ".")

list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS} -O3 -g -DNDEBUG")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS} -O3 -DNDEBUG")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS} -O0 -g -DDEBUG")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -lpthread")

SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${INCLUDES} -W -O3 -DNDEBUG")
SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS}" )

if(ENABLE_VC)
  include(VcMacros)
  vc_set_preferred_compiler_flags()
endif()

if (USE_AVX2)
  include_directories(${Vc_INCLUDE_DIR})
  string(REPLACE ";" " " VC_FLAGS "${Vc_COMPILE_FLAGS}")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${VC_FLAGS} -mavx2 -mfma -mbmi2")
endif()

configure_file(lib/generate/config.h.in lib/generate/config.h)

set(generate_SOURCES
  src/generate/generate.cpp
  src/generate/generate_scalar.cpp)

if (USE_AVX2)
  set(generate_SOURCES ${generate_SOURCES}
      src/generate/generate_avx2.cpp
      src/generate/vectorclass_ranvec1.cpp)
endif()

include_directories(${CMAKE_SOURCE_DIR}/lib)
include_directories(${CMAKE_SOURCE_DIR}/lib/generate)
include_directories(${CMAKE_BINARY_DIR}/lib/generate)

add_library(generate SHARED ${generate_SOURCES})

if (USE_AVX2)
target_link_libraries(generate PUBLIC ${Vc_LIBRARIES})
target_include_directories(generate
    PUBLIC ${CMAKE_BINARY_DIR}/include/vectorclass)
add_dependencies(generate vectorclass-install)
target_compile_definitions(generate PUBLIC "-DVc_IMPL=AVX2")
if (Vc_BUILTIN)
  add_dependencies(generate VC-install)
  target_link_directories(generate PUBLIC ${Vc_DESTDIR}/lib)
endif()
add_dependencies(generate vectorclass-install)

endif()

if(ENABLE_PYTHON)
  include(XTensor)

  include(pybind11Tools)
  pybind11_add_module(k40gen "src/k40gen/bindings.cpp")
  target_link_libraries(k40gen PUBLIC generate)

  find_package(pybind11)
  target_include_directories(k40gen PUBLIC ${PYBIND11_INCLUDE_DIRS})

  if (xtp_BUILTIN)
    add_dependencies(k40gen xtp-install)
    target_include_directories(k40gen PRIVATE ${CMAKE_BINARY_DIR}/external/include)
  endif()

  # xtensor external dependencies
  if (USE_AVX2)
    target_compile_definitions(k40gen PRIVATE "-DXTENSOR_ENABLE_XSIMD")
  endif()

  # pybind11 headers
  find_package(pybind11 REQUIRED)
  target_include_directories(k40gen PRIVATE ${pybind11_INCLUDE_DIRS})

  # numpy headers
  find_package(NumPy REQUIRED)
  target_include_directories(k40gen PRIVATE ${NUMPY_INCLUDE_DIRS})
endif()

if(ENABLE_TESTS)
  enable_testing()
  add_subdirectory(tests)
endif()
