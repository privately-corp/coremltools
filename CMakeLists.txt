cmake_minimum_required(VERSION 3.10.2)

set(CMAKE_OSX_ARCHITECTURES x86_64)
set(CMAKE_DISABLE_IN_SOURCE_BUILD ON)

project(coremltools)

if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
  message(FATAL_ERROR "
    Source directory '${PROJECT_SOURCE_DIR}' is the same
    as binary directory '${PROJECT_BINARY_DIR}'; coremltools requires
    an out-of-source build.  Note that your directory tree will require
    you to remove CMakeCache.txt before this will work, and CMake may have
    clobbered some source files (use git reset --hard).
    See: https://gitlab.kitware.com/cmake/community/wikis/FAQ#i-run-an-out-of-source-build-but-cmake-generates-in-source-anyway-why
  ")
endif()

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Globally ignore "no symbols" warnings during compilation
SET(CMAKE_CXX_ARCHIVE_CREATE "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
if(APPLE)
    SET(CMAKE_CXX_ARCHIVE_FINISH "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
endif()

find_program(HAS_CCACHE ccache)
if(HAS_CCACHE)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
endif()

add_subdirectory(deps)
add_subdirectory(mlmodel)

find_package(PythonInterp)
find_package(PythonLibs)

message("Found python at ${PYTHON_EXECUTABLE}")
message("Found python version ${PYTHON_VERSION_STRING}")
message("Found python includes ${PYTHON_INCLUDE_DIRS}")

include_directories(
  .
  deps/caffe/include
  deps/caffe/include/caffe/proto
  deps/protobuf/src
  deps/pybind11/include
  mlmodel/src
  ${PYTHON_INCLUDE_DIRS}
  )

if(APPLE)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fobjc-arc")
endif()

add_library(caffeconverter
  SHARED
  caffeconverter/CaffeConverterLib.cpp
  caffeconverter/CaffeConverterPython.cpp
  caffeconverter/Caffe/Pooling.cpp
  caffeconverter/Caffe/Embed.cpp
  caffeconverter/Caffe/Parameter.cpp
  caffeconverter/Caffe/Split.cpp
  caffeconverter/Caffe/Bias.cpp
  caffeconverter/Caffe/Reduction.cpp
  caffeconverter/Caffe/Batchnorm.cpp
  caffeconverter/Caffe/Slice.cpp
  caffeconverter/Caffe/Crop.cpp
  caffeconverter/Caffe/Concat.cpp
  caffeconverter/Caffe/InnerProduct.cpp
  caffeconverter/Caffe/LRN.cpp
  caffeconverter/Caffe/Power.cpp
  caffeconverter/Caffe/Scale.cpp
  caffeconverter/Caffe/MVN.cpp
  caffeconverter/Caffe/LSTM.cpp
  caffeconverter/Caffe/Abs.cpp
  caffeconverter/Caffe/Eltwise.cpp
  caffeconverter/Caffe/Exp.cpp
  caffeconverter/Caffe/InputLayers.cpp
  caffeconverter/Caffe/Softmax.cpp
  caffeconverter/Caffe/TrainingLayers.cpp
  caffeconverter/Caffe/CaffeConverter.cpp
  caffeconverter/Caffe/UpgradeProto.cpp
  caffeconverter/Caffe/Flatten.cpp
  caffeconverter/Caffe/Log.cpp
  caffeconverter/Caffe/Activation.cpp
  caffeconverter/Caffe/Convolution.cpp
  caffeconverter/Caffe/Reshape.cpp
  deps/caffe/src/caffe/proto/caffe.pb.cc
  deps/caffe/src/caffe/util/upgrade_proto.cpp
  )

target_compile_definitions(caffeconverter
  PRIVATE
  CPU_ONLY=1
  )

target_link_libraries(caffeconverter
  mlmodel
  libprotobuf
  )

if (APPLE)
  # Allow Python to be found at runtime instead of compile/link time
  # This is apparently the default on Linux
  set_target_properties(caffeconverter PROPERTIES LINK_FLAGS "-undefined dynamic_lookup")
endif()

file(COPY ${CMAKE_SOURCE_DIR}/README.md DESTINATION ${CMAKE_BINARY_DIR})
file(COPY ${CMAKE_SOURCE_DIR}/coremltools/__init__.py
  DESTINATION ${CMAKE_BINARY_DIR}/coremltools)
file(COPY ${CMAKE_SOURCE_DIR}/coremltools/__main__.py
  DESTINATION ${CMAKE_BINARY_DIR}/coremltools)
file(COPY ${CMAKE_SOURCE_DIR}/coremltools/version.py
  DESTINATION ${CMAKE_BINARY_DIR}/coremltools)

set(copy_dirs _deps _scripts converters models proto)
foreach(cdir IN ITEMS ${copy_dirs})
  file(COPY ${CMAKE_SOURCE_DIR}/coremltools/${cdir}
    DESTINATION ${CMAKE_BINARY_DIR}/coremltools)
endforeach()

if(NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
  set(_additional_caffeconverter_command COMMAND strip -x ${PROJECT_SOURCE_DIR}/coremltools/libcaffeconverter.so)
endif()

add_custom_command(
  TARGET caffeconverter
  POST_BUILD
  COMMAND cp $<TARGET_FILE:caffeconverter> ${PROJECT_SOURCE_DIR}/coremltools/libcaffeconverter.so
  ${_additional_caffeconverter_command}
)


find_library(CORE_VIDEO CoreVideo)
find_library(CORE_ML CoreML)
find_library(FOUNDATION Foundation)

if (APPLE AND CORE_VIDEO AND CORE_ML AND FOUNDATION)
  execute_process(
      COMMAND ${PYTHON_EXECUTABLE} -c "import numpy; print(numpy.get_include())"
      RESULT_VARIABLE NUMPY_INCLUDE_STATUS
      OUTPUT_VARIABLE NUMPY_INCLUDE
  )

  if("${NUMPY_INCLUDE}" STREQUAL "" OR NOT NUMPY_INCLUDE_STATUS EQUAL 0)
      message(FATAL_ERROR "Could not find numpy include path. Exit code: ${NUMPY_INCLUDE_STATUS}")
  endif()
  message("Found numpy include path at ${NUMPY_INCLUDE}")

  include_directories(
    ${NUMPY_INCLUDE}
  )

  add_library(coremlpython
    SHARED
    coremlpython/CoreMLPython.mm
    coremlpython/CoreMLPython.h
    coremlpython/CoreMLPythonArray.mm
    coremlpython/CoreMLPythonArray.h
    coremlpython/CoreMLPythonUtils.mm
    coremlpython/CoreMLPythonUtils.h
  )
  target_link_libraries(coremlpython
    mlmodel
    ${CORE_VIDEO}
    ${CORE_ML}
    ${FOUNDATION}
  )

if(APPLE)
  set(osx_export_file ${CMAKE_SOURCE_DIR}/coremlpython/exported_symbols_osx.ver)
  set_property(TARGET coremlpython APPEND PROPERTY LINK_DEPENDS "${osx_export_file}")
  set_property(TARGET coremlpython APPEND_STRING PROPERTY LINK_FLAGS " -Wl,-exported_symbols_list,${osx_export_file} ")

  # Allow Python to be found at runtime instead of compile/link time
  # This is apparently the default on Linux
  set_property(TARGET coremlpython APPEND_STRING PROPERTY LINK_FLAGS "-undefined dynamic_lookup")

else()
  set(linux_export_file coremlpython/exported_symbols_linux.ver)
  set_property(TARGET coremlpython APPEND_STRING PROPERTY LINK_FLAGS " -Wl,--version-script=${linux_export_file} ")
endif()

  set_property(TARGET coremlpython APPEND_STRING PROPERTY LINK_FLAGS " -Wl,-dead_strip")

  if(NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(_additional_libcoremlpython_command
      COMMAND strip -x ${PROJECT_SOURCE_DIR}/coremltools/libcoremlpython.so
    )
  endif()

  add_custom_command(
    TARGET coremlpython
    POST_BUILD
    COMMAND cp $<TARGET_FILE:coremlpython> ${PROJECT_SOURCE_DIR}/coremltools/libcoremlpython.so
    ${_additional_libcoremlpython_command}
  )

else()
  message(STATUS "CoreML.framework and dependent frameworks not found. Skipping libcoremlpython build.")
endif()

set(PYTHON_TAG "cp${PYTHON_VERSION_MAJOR}${PYTHON_VERSION_MINOR}")
if(APPLE)
  set(PLAT_NAME "macosx_10_16_intel;macosx_10_15_intel;macosx_10_14_intel;macosx_10_13_intel;macosx_10_12_intel")
elseif("${CMAKE_SYSTEM_NAME}" MATCHES "Linux")
  set(PLAT_NAME "manylinux1_x86_64")
else()
  message(FATAL_ERROR "Unsupported build platform. Supported platforms are Linux and macOS.")
endif()

# Add a target for each platform, and then a 'dist' that will build all of them.
# Parallel invocations of setup.py is not safe, so we serialize them.
set(plat_targets "")
foreach(platform IN ITEMS ${PLAT_NAME})
  add_custom_target(dist_${platform}
    COMMENT "Building dist for platform ${platform}..."
    COMMAND ${PYTHON_EXECUTABLE}
      ${CMAKE_SOURCE_DIR}/setup.py
      bdist_wheel
      --plat-name=${platform}
      --python-tag=${PYTHON_TAG}
      --dist-dir=${PROJECT_BINARY_DIR}/dist
    DEPENDS "caffeconverter;coremlpython;${plat_targets}"
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
  )
  set(plat_targets "${plat_targets};dist_${platform}")
endforeach()
# Add a 'dist' target that will build wheels for all possible platforms.
add_custom_target(dist DEPENDS ${plat_targets})

add_custom_target(pip_install_dev
  COMMAND pip install -e ${PROJECT_SOURCE_DIR}
  DEPENDS "caffeconverter;coremlpython"
)

add_custom_target(pytest
  COMMAND pytest -r fs ${PROJECT_SOURCE_DIR}/coremltools/test/ --timeout=600
  DEPENDS pip_install_dev
  USES_TERMINAL
)

add_custom_target(pytest_no_slow
  COMMAND pytest -r fs -m '"no slow"' ${PROJECT_SOURCE_DIR}/coremltools/test/
  DEPENDS pip_install_dev
  USES_TERMINAL
)
