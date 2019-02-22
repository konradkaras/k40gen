find_package(Catch2)

if(NOT Catch2_FOUND)
  # catch
  set(catch_VERSION "2.6.1")
  set(catch_SRC_URI "https://github.com/catchorg/Catch2/releases/download/v${catch_VERSION}/catch.hpp")
  set(catch_DESTDIR "${CMAKE_BINARY_DIR}/include")
  set(catch_ROOTDIR "${catch_DESTDIR}/catch2")

  ExternalProject_Add(catch
    URL ${catch_SRC_URI}
    URL_HASH SHA256=a53ef31cf0bd4c1038cdaf8a38d6c88ee762e6aba181f9534a026363bc73e430
    SOURCE_DIR catch2
    INSTALL_DIR ${catch_ROOTDIR}
    CONFIGURE_COMMAND ""
    INSTALL_COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR> <INSTALL_DIR>
    STEP_TARGETS install)
  set(Catch2_FOUND TRUE)
  set(builtin_catch TRUE)
else()
  set(builtin_catch FALSE)
endif()

include_directories(include)

if (USE_AVX2)
  add_definitions("-DUSE_AVX2")
endif()

add_library(test_functions STATIC
  lib/coincidences.cpp
  lib/generate.cpp)

add_executable(test_k40gen
  src/test_coincidences.cpp
  src/test_rates.cpp
  src/test_main.cpp)

target_link_libraries(test_k40gen PRIVATE generate test_functions)

add_test(TestK40 test_k40gen)

find_package(ROOT COMPONENTS Core Cling Hist Gpad)

add_executable(test_k40gen_root
  src/test_rates_root.cpp)
target_include_directories(test_k40gen_root PRIVATE
  ${ROOT_INCLUDE_DIRS})

target_link_libraries(test_k40gen_root PRIVATE
  generate
  test_functions
  ${ROOT_LIBRARIES})