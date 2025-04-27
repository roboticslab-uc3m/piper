cmake_minimum_required(VERSION 3.13)

project(piper C CXX)

file(READ "${CMAKE_CURRENT_LIST_DIR}/VERSION" piper_version)
string(STRIP ${piper_version} piper_version)

message(STATUS "Piper version: ${piper_version}")

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# https://gitlab.kitware.com/cmake/community/-/wikis/doc/cmake/RPATH-handling#always-full-rpath
set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")

include(GNUInstallDirs)

if(MSVC)
  # Force compiler to use UTF-8 for IPA constants
  add_compile_options("$<$<C_COMPILER_ID:MSVC>:/utf-8>")
  add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")
elseif(NOT APPLE)
  # Linux flags
  string(APPEND CMAKE_CXX_FLAGS " -Wall -Wextra -Wl,-rpath,'$ORIGIN'")
  string(APPEND CMAKE_C_FLAGS " -Wall -Wextra")
endif()

# Required in roboticslab-uc3m projects
find_package(YCM REQUIRED)

add_library(piper_lib SHARED src/cpp/piper.cpp)
add_executable(piper src/cpp/main.cpp)
add_executable(test_piper src/cpp/test.cpp)

# NOTE: external project prefix are shortened because of path length restrictions on Windows
# NOTE: onnxruntime is pulled from piper-phonemize

# ---- fmt ---

if(NOT DEFINED FMT_DIR)
  set(FMT_VERSION "10.0.0")
  set(FMT_DIR "${CMAKE_CURRENT_BINARY_DIR}/fi")

  include(ExternalProject)
  ExternalProject_Add(
    fmt_external
    PREFIX "${CMAKE_CURRENT_BINARY_DIR}/f"
    URL "https://github.com/fmtlib/fmt/archive/refs/tags/${FMT_VERSION}.zip"
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=${FMT_DIR}
    CMAKE_ARGS -DFMT_TEST:BOOL=OFF  # Don't build all the tests
  )
  add_dependencies(piper_lib fmt_external)
endif()

# ---- spdlog ---

if(NOT DEFINED SPDLOG_DIR)
  set(SPDLOG_DIR "${CMAKE_CURRENT_BINARY_DIR}/si")
  set(SPDLOG_VERSION "1.12.0")
  ExternalProject_Add(
    spdlog_external
    PREFIX "${CMAKE_CURRENT_BINARY_DIR}/s"
    URL "https://github.com/gabime/spdlog/archive/refs/tags/v${SPDLOG_VERSION}.zip"
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=${SPDLOG_DIR}
  )
  add_dependencies(piper_lib spdlog_external)
endif()

# ---- piper-phonemize ---

if(NOT DEFINED PIPER_PHONEMIZE_DIR)
  set(PIPER_PHONEMIZE_DIR "${CMAKE_CURRENT_BINARY_DIR}/pi")
  ExternalProject_Add(
    piper_phonemize_external
    PREFIX "${CMAKE_CURRENT_BINARY_DIR}/p"
    URL "https://github.com/roboticslab-uc3m/piper-phonemize/archive/refs/heads/rl.zip"
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=${PIPER_PHONEMIZE_DIR}
  )
  add_dependencies(piper_lib piper_phonemize_external)
endif()

# ---- Declare targets ----

if((NOT MSVC) AND (NOT APPLE))
  # Linux flags
  string(APPEND CMAKE_CXX_FLAGS " -Wall -Wextra")
  string(APPEND CMAKE_C_FLAGS " -Wall -Wextra")

  set(PIPER_EXTRA_LIBRARIES "pthread")
endif()

target_link_libraries(piper_lib
  PUBLIC
    piper_phonemize
    onnxruntime
  PRIVATE
    fmt
    spdlog
    espeak-ng
    ${PIPER_EXTRA_LIBRARIES}
)

target_link_libraries(piper piper_lib fmt spdlog)
target_link_libraries(test_piper piper_lib fmt spdlog)

target_compile_features(piper_lib PUBLIC cxx_std_17)

set_property(TARGET piper_lib PROPERTY
  PUBLIC_HEADER
    include/piper/piper.hpp
    include/piper/json.hpp)

target_link_directories(piper_lib
  PUBLIC
    $<BUILD_INTERFACE:${PIPER_PHONEMIZE_DIR}/lib>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_LIBDIR}>
)

target_include_directories(piper_lib
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
    $<BUILD_INTERFACE:${PIPER_PHONEMIZE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

target_compile_definitions(piper_lib PRIVATE _PIPER_VERSION=${piper_version})
target_compile_definitions(piper PRIVATE _PIPER_SHARE_DIR="${CMAKE_INSTALL_FULL_DATADIR}")

# ---- Declare test ----
include(CTest)
enable_testing()
add_test(
  NAME test_piper
  COMMAND test_piper "${CMAKE_SOURCE_DIR}/etc/test_voice.onnx" "${PIPER_PHONEMIZE_DIR}/share/espeak-ng-data" "${CMAKE_CURRENT_BINARY_DIR}/test.wav"
)

# ---- Declare install targets ----

include(CMakePackageConfigHelpers)

set(PIPER_DATA_DIR "${PIPER_PHONEMIZE_DIR}/share")

configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/piperConfig.cmake"
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/piper
  PATH_VARS PIPER_DATA_DIR
  INSTALL_PREFIX ${CMAKE_BINARY_DIR}
)

set(PIPER_DATA_DIR ${CMAKE_INSTALL_DATADIR})

configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/piperConfigInstall.cmake"
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/piper
  PATH_VARS PIPER_DATA_DIR
)

write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/piperConfigVersion.cmake"
  VERSION "${piper_version}"
  COMPATIBILITY AnyNewerVersion
)

install(
  TARGETS piper_lib
  EXPORT piper
  PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/piper)

install(TARGETS piper)

export(
  TARGETS piper_lib
  NAMESPACE piper::
  FILE piperTargets.cmake)

install(
  EXPORT piper
  NAMESPACE piper::
  FILE piperTargets.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/piper)

install(
  FILES "${CMAKE_CURRENT_BINARY_DIR}/piperConfigInstall.cmake"
  RENAME piperConfig.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/piper
)

install(
  FILES "${CMAKE_CURRENT_BINARY_DIR}/piperConfigVersion.cmake"
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/piper
)

# Dependencies
install(
  DIRECTORY ${PIPER_PHONEMIZE_DIR}/bin/
  DESTINATION ${CMAKE_INSTALL_BINDIR}
  USE_SOURCE_PERMISSIONS  # keep +x
  FILES_MATCHING
  PATTERN "piper_phonemize"
  PATTERN "espeak-ng"
  PATTERN "*.dll"
)

install(
  DIRECTORY ${PIPER_PHONEMIZE_DIR}/include/
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

install(
  DIRECTORY ${PIPER_PHONEMIZE_DIR}/lib/
  DESTINATION ${CMAKE_INSTALL_LIBDIR}
  FILES_MATCHING
  PATTERN "*.dll"
  PATTERN "*.so*"
)

install(
  DIRECTORY ${PIPER_PHONEMIZE_DIR}/share/espeak-ng-data
  DESTINATION ${CMAKE_INSTALL_DATADIR}
)

install(
  FILES ${PIPER_PHONEMIZE_DIR}/share/libtashkeel_model.ort
  DESTINATION ${CMAKE_INSTALL_DATADIR}
)

# Added by YCM
include(AddUninstallTarget)
