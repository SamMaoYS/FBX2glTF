cmake_minimum_required(VERSION 3.18)
project(FBX2glTF)

set(typical_usage_str
  "Example usage:\n\
  > mkdir build\n\
  > cd build && cmake .. \n\
  > make && make install")

if ("${CMAKE_CURRENT_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
  message(FATAL_ERROR
    "Building from within the source tree is not supported! ${typical_usage_str}")
endif ()

set(CMAKE_CXX_STANDARD 11)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}")
include(ExternalProject)

option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(BUILD_PYTHON_LIBS "Build Python libraries" ON)

# https://blog.kitware.com/cmake-and-the-default-build-type/
if(NOT CMAKE_CONFIGURATION_TYPES)
  if(NOT CMAKE_BUILD_TYPE)
    message(STATUS "Setting build type to Release as none was specified.")
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
    # Set the possible values of build type for cmake-gui.
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
      "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
  endif()
  message(STATUS "CMAKE_BUILD_TYPE is set to ${CMAKE_BUILD_TYPE}.")
endif()

message(STATUS "----------------------------------------")
message(STATUS "BUILD_SHARED_LIBS: ${BUILD_SHARED_LIBS}")
message(STATUS "BUILD_PYTHON_LIBS: ${BUILD_PYTHON_LIBS}")
message(STATUS "")
message(STATUS "CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}")
message(STATUS "CMAKE_INSTALL_PREFIX: ${CMAKE_INSTALL_PREFIX}")
message(STATUS "CMAKE_PREFIX_PATH: ${CMAKE_PREFIX_PATH}")
message(STATUS "----------------------------------------")

set(CMAKE_CORE_BUILD_FLAGS
  -DCMAKE_PREFIX_PATH=${CMAKE_PREFIX_PATH}
  -DCMAKE_INCLUDE_PATH=${CMAKE_INCLUDE_PATH}
  -DCMAKE_LIBRARY_PATH=${CMAKE_LIBRARY_PATH}
  -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
  -DBUILD_SHARED_LIBS:BOOL=${BUILD_SHARED_LIBS}
  -DCMAKE_INSTALL_DO_STRIP:BOOL=${CMAKE_INSTALL_DO_STRIP}
  -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
  -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
  )

# FBX
foreach (FBXSDK_VERSION "2020.3")
  find_package(FBX)
  if (FBXSDK_FOUND)
    break()
  endif ()
endforeach (FBXSDK_VERSION)
if (NOT FBXSDK_FOUND)
  message(FATAL_ERROR
    "Can't find FBX SDK in either:\n"
    " - Mac OS X: ${FBXSDK_APPLE_ROOT}\n"
    " - Windows: ${FBXSDK_WINDOWS_ROOT}\n"
    " - Linux: ${FBXSDK_LINUX_ROOT}"
    )
endif ()

set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
find_package(Threads REQUIRED)

list(INSERT CMAKE_MODULE_PATH 0 "${CMAKE_BINARY_DIR}")

# Libraries we get from conda
find_package(Boost 1.69 REQUIRED COMPONENTS filesystem)
find_package(LibXml2 REQUIRED)
find_package(ZLIB REQUIRED)
find_package(fmt REQUIRED)
find_package(PythonLibs REQUIRED)
find_package(Python 3.6 COMPONENTS Interpreter Development REQUIRED)
find_package(pybind11 CONFIG REQUIRED)
if (APPLE)
  find_package(Iconv MODULE REQUIRED)
else ()
  find_package(Iconv REQUIRED)
endif ()

# create a compilation database for e.g. clang-tidy
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# DRACO
ExternalProject_Add(Draco
  GIT_REPOSITORY https://github.com/google/draco
  GIT_TAG 1.5.3
  PREFIX draco
  INSTALL_DIR ${CMAKE_INSTALL_PREFIX}
  CMAKE_ARGS
  ${CMAKE_CORE_BUILD_FLAGS}
  -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
  -DBUILD_FOR_GLTF=1
  )
set(DRACO_INCLUDE_DIR "${CMAKE_INSTALL_PREFIX}/include")
if (WIN32)
  set(DRACO_LIB "${CMAKE_INSTALL_PREFIX}/lib/dracoenc.lib")
else ()
  set(DRACO_LIB "${CMAKE_INSTALL_PREFIX}/lib/libdraco.so")
endif ()

# MATHFU
set(mathfu_build_benchmarks OFF CACHE BOOL "")
set(mathfu_build_tests OFF CACHE BOOL "")
ExternalProject_Add(MathFu
  PREFIX mathfu
  GIT_REPOSITORY https://github.com/google/mathfu
  GIT_TAG v1.1.0
  CONFIGURE_COMMAND ${CMAKE_COMMAND} -E echo "Skipping MathFu configure step."
  BUILD_COMMAND ${CMAKE_COMMAND} -E echo "Skipping MathFu build step."
  INSTALL_COMMAND ${CMAKE_COMMAND} -E echo "Skipping MathFu install step."
  )
set(MATHFU_INCLUDE_DIRS
  "${CMAKE_BINARY_DIR}/mathfu/src/MathFu/include/"
  "${CMAKE_BINARY_DIR}/mathfu/src/MathFu/dependencies/vectorial/include")

# OrderedMap
ExternalProject_Add(FiFoMap
  PREFIX fifo_map
  GIT_REPOSITORY https://github.com/nlohmann/fifo_map
  CONFIGURE_COMMAND ${CMAKE_COMMAND} -E echo "Skipping FiFoMap configure step."
  BUILD_COMMAND ${CMAKE_COMMAND} -E echo "Skipping FiFoMap build step."
  INSTALL_COMMAND ${CMAKE_COMMAND} -E echo "Skipping FiFoMap install step."
  )
set(FIFO_MAP_INCLUDE_DIR "${CMAKE_BINARY_DIR}/fifo_map/src/FiFoMap/src")


# cppcodec
ExternalProject_Add(CPPCodec
  PREFIX cppcodec
  GIT_REPOSITORY https://github.com/tplgy/cppcodec
  CONFIGURE_COMMAND ${CMAKE_COMMAND} -E echo "Skipping CPPCodec configure step."
  BUILD_COMMAND ${CMAKE_COMMAND} -E echo "Skipping CPPCodec build step."
  INSTALL_COMMAND ${CMAKE_COMMAND} -E echo "Skipping CPPCodec install step."
  )
set(CPPCODEC_INCLUDE_DIR "${CMAKE_BINARY_DIR}/cppcodec/src/CPPCodec")

if (APPLE)
  find_library(CF_FRAMEWORK CoreFoundation)
  message("CoreFoundation Framework: ${CF_FRAMEWORK}")
  set(FRAMEWORKS ${CF_FRAMEWORK})
endif ()

set(LIB_SOURCE_FILES
  src/FBX2glTF.h
  src/fbx/materials/3dsMaxPhysicalMaterial.cpp
  src/fbx/materials/FbxMaterials.cpp
  src/fbx/materials/FbxMaterials.hpp
  src/fbx/materials/RoughnessMetallicMaterials.hpp
  src/fbx/materials/StingrayPBSMaterial.cpp
  src/fbx/materials/TraditionalMaterials.cpp
  src/fbx/materials/TraditionalMaterials.hpp
  src/fbx/Fbx2Raw.cpp
  src/fbx/Fbx2Raw.hpp
  src/fbx/FbxBlendShapesAccess.cpp
  src/fbx/FbxBlendShapesAccess.hpp
  src/fbx/FbxLayerElementAccess.hpp
  src/fbx/FbxSkinningAccess.cpp
  src/fbx/FbxSkinningAccess.hpp
  src/gltf/Raw2Gltf.cpp
  src/gltf/Raw2Gltf.hpp
  src/gltf/GltfModel.cpp
  src/gltf/GltfModel.hpp
  src/gltf/TextureBuilder.cpp
  src/gltf/TextureBuilder.hpp
  src/gltf/properties/AccessorData.cpp
  src/gltf/properties/AccessorData.hpp
  src/gltf/properties/AnimationData.cpp
  src/gltf/properties/AnimationData.hpp
  src/gltf/properties/BufferData.cpp
  src/gltf/properties/BufferData.hpp
  src/gltf/properties/BufferViewData.cpp
  src/gltf/properties/BufferViewData.hpp
  src/gltf/properties/CameraData.cpp
  src/gltf/properties/CameraData.hpp
  src/gltf/properties/ImageData.cpp
  src/gltf/properties/ImageData.hpp
  src/gltf/properties/LightData.cpp
  src/gltf/properties/LightData.hpp
  src/gltf/properties/MaterialData.cpp
  src/gltf/properties/MaterialData.hpp
  src/gltf/properties/MeshData.cpp
  src/gltf/properties/MeshData.hpp
  src/gltf/properties/NodeData.cpp
  src/gltf/properties/NodeData.hpp
  src/gltf/properties/PrimitiveData.cpp
  src/gltf/properties/PrimitiveData.hpp
  src/gltf/properties/SamplerData.hpp
  src/gltf/properties/SceneData.cpp
  src/gltf/properties/SceneData.hpp
  src/gltf/properties/SkinData.cpp
  src/gltf/properties/SkinData.hpp
  src/gltf/properties/TextureData.cpp
  src/gltf/properties/TextureData.hpp
  src/mathfu.hpp
  src/raw/RawModel.cpp
  src/raw/RawModel.hpp
  src/utils/File_Utils.cpp
  src/utils/File_Utils.hpp
  src/utils/Image_Utils.cpp
  src/utils/Image_Utils.hpp
  src/utils/String_Utils.hpp
  third_party/CLI11/CLI11.hpp
  )

add_library(libFBX2glTF SHARED ${LIB_SOURCE_FILES})
set_target_properties(libFBX2glTF PROPERTIES OUTPUT_NAME "FBX2glTF")
add_executable(appFBX2glTF src/FBX2glTF.cpp)
set_target_properties(appFBX2glTF PROPERTIES OUTPUT_NAME "FBX2glTF")

add_dependencies(libFBX2glTF
  Draco
  MathFu
  FiFoMap
  CPPCodec
  )

if (NOT MSVC)
  # Disable annoying & spammy warning from FBX SDK header file
  target_compile_options(libFBX2glTF PUBLIC
    "-Wno-null-dereference"
    "-Wunused"
    )
  target_compile_options(appFBX2glTF PUBLIC
    "-Wno-null-dereference"
    "-Wunused"
    )
endif ()

target_link_libraries(libFBX2glTF PUBLIC
  ${FRAMEWORKS}
  ${Boost_LIBRARIES}
  ${DRACO_LIB}
  optimized ${FBXSDK_LIBRARY}
  debug ${FBXSDK_LIBRARY_DEBUG}
  fmt::fmt
  ${LIBXML2_LIBRARIES}
  ZLIB::ZLIB
  ${PYTHON_LIBRARIES}
  ${CMAKE_DL_LIBS}
  ${CMAKE_THREAD_LIBS_INIT}
  )

if (APPLE)
  target_link_libraries(libFBX2glTF PUBLIC Iconv)
else ()
  target_link_libraries(libFBX2glTF PUBLIC ${Iconv_LIBRARIES})
endif ()

target_include_directories(libFBX2glTF PUBLIC
  ${CMAKE_CURRENT_SOURCE_DIR}/src
  )

target_include_directories(libFBX2glTF SYSTEM PUBLIC
  "third_party/stb"
  "third_party/json"
  ${FBXSDK_INCLUDE_DIR}
  ${DRACO_INCLUDE_DIR}
  ${MATHFU_INCLUDE_DIRS}
  ${FIFO_MAP_INCLUDE_DIR}
  ${CPPCODEC_INCLUDE_DIR}
  ${PYTHON_INCLUDE_DIRS}
  )

target_include_directories(appFBX2glTF PUBLIC
  "third_party/CLI11"
  )
target_link_libraries(appFBX2glTF PUBLIC libFBX2glTF)

if (BUILD_PYTHON_LIBS)
  pybind11_add_module(PyFBX2glTF src/FBX2glTF.cpp ${LIB_SOURCE_FILES})

  add_dependencies(PyFBX2glTF
    Draco
    MathFu
    FiFoMap
    CPPCodec
    )

  target_link_libraries(PyFBX2glTF PUBLIC
    ${FRAMEWORKS}
    ${Boost_LIBRARIES}
    ${DRACO_LIB}
    optimized ${FBXSDK_LIBRARY}
    debug ${FBXSDK_LIBRARY_DEBUG}
    fmt::fmt
    ${LIBXML2_LIBRARIES}
    ZLIB::ZLIB
    ${PYTHON_LIBRARIES}
    ${CMAKE_DL_LIBS}
    ${CMAKE_THREAD_LIBS_INIT}
    )

  if (APPLE)
    target_link_libraries(PyFBX2glTF PUBLIC Iconv)
  else ()
    target_link_libraries(PyFBX2glTF PUBLIC ${Iconv_LIBRARIES})
  endif ()

  target_include_directories(PyFBX2glTF PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    "third_party/CLI11"
    )

  target_include_directories(PyFBX2glTF PUBLIC
    "third_party/stb"
    "third_party/json"
    ${FBXSDK_INCLUDE_DIR}
    ${DRACO_INCLUDE_DIR}
    ${MATHFU_INCLUDE_DIRS}
    ${FIFO_MAP_INCLUDE_DIR}
    ${CPPCODEC_INCLUDE_DIR}
    ${PYTHON_INCLUDE_DIRS}
    )
endif()

install(TARGETS libFBX2glTF appFBX2glTF PyFBX2glTF
  RUNTIME DESTINATION bin
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  )
