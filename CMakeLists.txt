project(GraphLab)

# We require the most recent version of cmake and automatically
# install the correct version when using the cmake lists
cmake_minimum_required(VERSION 2.8)

# Libraries linked via full path no longer produce linker search paths.
cmake_policy(SET CMP0003 NEW)
# Preprocessor definition values are now escaped automatically.
cmake_policy(SET CMP0005 NEW)

# Determine where additional GraphLab specific cmake modules are
# defined
set(CMAKE_MODULE_PATH ${GraphLab_SOURCE_DIR}/cmake)

# TODO: what is this used by
set(BUILD_SHARED_LIBS ON)

include(CheckLibraryExists)
include(CheckFunctionExists)
include(ExternalProject)
include(CheckCXXSourceCompiles)

# check we are on a 64 bit machine (else fail)
if( CMAKE_SIZEOF_VOID_P EQUAL 4 )
   message( SEND_ERROR "GraphLab does not support 32 bit systems. Please switch to 64 bit system and try again" )
endif ( CMAKE_SIZEOF_VOID_P EQUAL 4 )


# set include path
include_directories(
  ${GraphLab_SOURCE_DIR}/src
  ${GraphLab_SOURCE_DIR}/cxxtest)


add_definitions(-DUSE_DYNAMIC_LOCAL_GRAPH)

if(NO_OPENMP)
  set(OPENMP_C_FLAGS "")
  set(OPENMP_LIBRARIES "")
  add_definitions(-D__NO_OPENMP__)
else()
  set(OPENMP_C_FLAGS "-fopenmp")
  set(OPENMP_LIBRARIES "gomp")
endif()

if (APPLE)
link_libraries(pthread ${OPENMP_LIBRARIES})
else()
link_libraries(pthread rt ${OPENMP_LIBRARIES})
endif()

set(CPP11_FLAGS "")
if(CPP11)
  message(STATUS "C++11 Enabled")
  set(CPP11_FLAGS "-std=c++11 -Wno-enum-compare -Wno-conversion-null")
endif()

if(VID32)
  message(STATUS "Using 32bit vertex id types")
  add_definitions(-DUSE_VID32)
endif()


# Shared compiler flags used by all builds (debug, profile, release)
set(COMPILER_FLAGS "-Wall -g ${CPP11_FLAGS} ${OPENMP_C_FLAGS}" CACHE STRING "common compiler options")

# Disable address space randomization for OSX lion and above
if (APPLE)
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Xlinker -no_pie")
endif()

set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-rpath")

# Set subdirectories
subdirs(src tests demoapps toolkits)
if(EXPERIMENTAL)
  if (IS_DIRECTORY ${GraphLab_SOURCE_DIR}/experimental)
    subdirs(experimental)
  endif()
endif()

if (IS_DIRECTORY ${GraphLab_SOURCE_DIR}/apps)
  subdirs(apps)
endif()

# Set installation rules
install(DIRECTORY src/
  DESTINATION include
  FILES_MATCHING PATTERN "*.hpp"  PATTERN "*.h"
  PATTERN ".svn" EXCLUDE
  )
install(CODE "execute_process(COMMAND ./scripts/install_deps.sh
  ${CMAKE_INSTALL_PREFIX} WORKING_DIRECTORY
  ${CMAKE_CURRENT_SOURCE_DIR})")


add_definitions(-DHAS_EIGEN)

macro(requires_eigen NAME)
  add_dependencies(${NAME} eigen)
endmacro(requires_eigen)


message(STATUS "Boost libs: " ${Boost_LIBRARIES})
foreach(blib ${Boost_LIBRARIES})
  message(STATUS "Boost libs: " ${blib})
  string(REGEX REPLACE "\\.a$" ${CMAKE_SHARED_LIBRARY_SUFFIX} bout ${blib})
  message(STATUS "Boost dyn libs: " ${bout})
  set(Boost_SHARED_LIBRARIES ${Boost_SHARED_LIBRARIES} ${bout})
endforeach()
message(STATUS "Boost Shared libs: " ${Boost_SHARED_LIBRARIES})

if(NOT NO_TCMALLOC)
    if(APPLE)
    set (tcmalloc_shared "--enable-shared=yes")
    else()
    set (tcmalloc_shared "--enable-shared=no")
    endif()
    set(TCMALLOC-FOUND 1)
    add_definitions(-DHAS_TCMALLOC)
endif()


macro(requires_opencv NAME)
  get_target_property(tmp ${NAME} COMPILE_FLAGS)
  set_target_properties(${NAME} PROPERTIES COMPILE_FLAGS "${tmp}")
  target_link_libraries(${NAME} opencv_core opencv_imgproc opencv_calib3d opencv_video opencv_features2d opencv_ml opencv_highgui opencv_objdetect opencv_contrib opencv_legacy opencv_contrib opencv_stitching)
  add_dependencies(${NAME} opencv)
endmacro(requires_opencv)

if (APPLE)
  ExternalProject_Add(libevent
    PREFIX ${GraphLab_SOURCE_DIR}/deps/event
    URL http://iweb.dl.sourceforge.net/project/levent/libevent/libevent-2.0/libevent-2.0.18-stable.tar.gz
    URL_MD5 aa1ce9bc0dee7b8084f6855765f2c86a
    CONFIGURE_COMMAND <SOURCE_DIR>/configure --prefix=<INSTALL_DIR> --disable-openssl --enable-shared=no
    INSTALL_DIR ${GraphLab_SOURCE_DIR}/deps/local
  )
else()
  link_libraries(event)
  link_libraries(event_pthreads)
endif()

if(NOT NO_JAVAC)
  find_package(Java)
  find_package(JNI)


  if((EXISTS ${JAVA_INCLUDE_PATH}/jni.h))
    set(JNI_REALLY_FOUND 1 CACHE BOOL "Tests whether the header actually exists")
    message(STATUS "jni.h was found at " ${JAVA_INCLUDE_PATH}/jni.h)
    include_directories(${JNI_INCLUDE_DIRS})
    ## Guess java home location
    if(EXISTS $ENV{JAVA_HOME} )
      set(JAVA_HOME $ENV{JAVA_HOME} CACHE STRING "Location of Java Home")
      message(STATUS "Java home set by user: " ${JAVA_HOME})
    else()
      string(REGEX REPLACE "\\/include" "" JAVA_HOME_GUESS  ${JAVA_AWT_INCLUDE_PATH})
      set(JAVA_HOME ${JAVA_HOME_GUESS} CACHE STRING "Location of Java Home")
      message(STATUS "Java Home guessed: " ${JAVA_HOME})
    endif()
  else( )
    set(JNI_REALLY_FOUND 0 CACHE BOOL "Tests whether the header actually exists")
    message(STATUS "jni.h was not found at " ${JAVA_INCLUDE_PATH}/jni.h)
  endif( )

  # Ant =========================================================================
  # Ant is used to build the graphlab JVM interface
  find_package(Ant)

  # LibHDFS =====================================================================
  # If JNI is found we install libhdfs which allows programs to read and write
  # to hdfs filesystems
  if( JNI_REALLY_FOUND )
    message(STATUS "Building libhdfs")
    set(HADOOP_FOUND 1 CACHE BOOL "Hadoop was found")
   # add_definitions(-DHAS_HADOOP)
  endif( JNI_REALLY_FOUND )

endif()



# MPI =========================================================================
# MPI is currently used to establish communication between rpc nodes
#sgr this name is confusing. i like WITH_MPI better. likewise for NO_OPENMP
if(NO_MPI)
  message(STATUS "MPI disabled as a configure option.")
else()
  #find_package(MPICH2)
  find_package(MPI) 
  if(MPI_FOUND)
    message(STATUS "MPI Found: " ${MPI_LIBRARY} ${MPI_EXTRA_LIBRARY})
    add_definitions(-DHAS_MPI)
    include_directories(${MPI_INCLUDE_PATH})
  else(MPI_FOUND)
    message(STATUS "MPI Not Found! Distributed Executables will not be compiled")
    set(MPI_LIBRARY "")
    set(MPI_EXTRA_LIBRARY "")
  endif(MPI_FOUND)
endif()


## =============================================================
# Core graphlab requirements
macro(requires_core_deps NAME)
  target_link_libraries(${NAME}
    ${Boost_LIBRARIES}
    z
    event event_pthreads
    zookeeper_mt)
  add_dependencies(${NAME} boost libevent zookeeper)
  if(MPI_FOUND)
    target_link_libraries(${NAME} ${MPI_LIBRARY} ${MPI_EXTRA_LIBRARY})
  endif(MPI_FOUND)
  if(HADOOP_FOUND)
    target_link_libraries(${NAME} hdfs ${JAVA_JVM_LIBRARY})
    add_dependencies(${NAME} hadoop)
  endif(HADOOP_FOUND)
  if(NOT NO_TCMALLOC)
    target_link_libraries(${NAME} tcmalloc)
    add_dependencies(${NAME} libtcmalloc)
  endif()
endmacro(requires_core_deps)

## ============================================================================
# System Checks

## ============================================================================
# Test for cpu affinity support
set(CMAKE_REQUIRED_LIBRARIES "pthread")
check_function_exists(pthread_setaffinity_np HAS_SET_AFFINITY)
set(CMAKE_REQUIRED_LIBRARIES ${crlbackup})

include(CheckCXXCompilerFlag)
## ============================================================================
# check if MARCH is set
if (NOT MARCH)
  #set to native if supported
  check_cxx_compiler_flag(-march=native HAS_MARCH_NATIVE)
  if(HAS_MARCH_NATIVE)
    set(MARCH "native")
  else()
    set(MARCH "x86-64")
  endif()
endif()

# check if MTUNE is set
if (NOT MTUNE)
  #set to native if supported
  check_cxx_compiler_flag(-mtune=native HAS_MTUNE_NATIVE)
  if(HAS_MTUNE_NATIVE)
    set(MTUNE "native")
  else()
    set(MTUNE "x86-64")
  endif()
endif()

## ============================================================================
# Setup compiler

# If profiling mode is enabled then additional profiling flags are set for
# the compiler
if (COMPILE_PROFILING MATCHES 1)
  set(PROFILING_FLAGS "-DUSE_EVENT_LOG -DUSE_TRACEPOINT")
else()
  set(PROFILING_FLAGS "")
endif()


#disable Wno-unused-local-typedefs if available
check_cxx_compiler_flag(-Wno-unused-local-typedefs HAS_WNO_LOCAL_TYPEDEFS)
if(HAS_WNO_LOCAL_TYPEDEFS)
  set(WNO_LOCAL_TYPEDEFS "-Wno-unused-local-typedefs")
else()
  set(WNO_LOCAL_TYPEDEFS "")
endif()


# Set the debug flags
set(CMAKE_C_FLAGS_DEBUG
  "-O0 -Wno-attributes -march=${MARCH} -Winit-self ${PROFILING_FLAGS} ${COMPILER_FLAGS}"
  CACHE STRING "compiler options" FORCE)
set(CMAKE_CXX_FLAGS_DEBUG
  "-O0 ${WNO_LOCAL_TYPEDEFS} -Wno-attributes -march=${MARCH} -Winit-self ${PROFILING_FLAGS} ${COMPILER_FLAGS}"
  CACHE STRING "compiler options" FORCE)

set(CMAKE_C_FLAGS_RELEASE
  "-O3 -Wno-attributes -march=${MARCH} -mtune=${MTUNE} ${PROFILING_FLAGS} ${COMPILER_FLAGS}"
  CACHE STRING "compiler options" FORCE)
set(CMAKE_CXX_FLAGS_RELEASE
  "-O3 ${WNO_LOCAL_TYPEDEFS} -Wno-attributes -march=${MARCH} -mtune=${MTUNE} ${PROFILING_FLAGS} ${COMPILER_FLAGS}"
  CACHE STRING "compiler options" FORCE)

set(CMAKE_REQUIRED_FLAGS ${CMAKE_CXX_FLAGS_RELEASE})
check_cxx_source_compiles("int main(int argc, char** argv) { return __builtin_ia32_crc32di(0,0);}" HAS_CRC32)
set(CMAKE_REQUIRED_FLAGS "")
if(HAS_CRC32)
  set(CMAKE_CXX_FLAGS_RELEASE
      "${CMAKE_CXX_FLAGS_RELEASE} -DHAS_BUILTIN_CRC32"
      CACHE STRING "compiler options" FORCE)
endif()

# check for gcc 4.8

if (CMAKE_BUILD_TYPE MATCHES "Release")
  message(STATUS "Release build with C++ flags: " ${CMAKE_CXX_FLAGS_RELEASE})
  message(STATUS "Release build with C flags: " ${CMAKE_C_FLAGS_RELEASE})
elseif(CMAKE_BUILD_TYPE MATCHES "Debug")
  message(STATUS "Debug build with C++ flags: " ${CMAKE_CXX_FLAGS_DEBUG})
  message(STATUS "Debug build with C flags: " ${CMAKE_C_FLAGS_DEBUG})
elseif(CMAKE_BUILD_TYPE MATCHES "Mex")
  message(STATUS "Mex CXX" ${CMAKE_CXX_FLAGS_MEX})
  message(STATUS "Mex C" ${CMAKE_C_FLAGS_MEX})
else()
  message(WARNING "Unknown build type: " ${CMAKE_BUILD_TYPE} "!")
endif()


## ============================================================================
## ============================================================================
## ============================================================================
# Setup testing tools
# Make sure testing is enabled
enable_testing()
# Use Python interpreter
find_package(PythonInterp)
set(CXXTESTGEN ${CMAKE_SOURCE_DIR}/cxxtest/cxxtestgen)
# create a macro to define a test
macro(ADD_CXXTEST NAME)
  if(PYTHONINTERP_FOUND)
    add_custom_command(
      OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${NAME}.cpp
      COMMAND
      ${PYTHON_EXECUTABLE} ${CXXTESTGEN}
      --runner=ErrorPrinter
      -o ${CMAKE_CURRENT_BINARY_DIR}/${NAME}.cpp ${ARGV}
      DEPENDS ${ARGV}
      WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
      )
  endif(PYTHONINTERP_FOUND)
  add_graphlab_executable(${NAME}test ${CMAKE_CURRENT_BINARY_DIR}/${NAME}.cpp)
  set_source_files_properties( ${CMAKE_CURRENT_BINARY_DIR}/${NAME}.cpp
    PROPERTIES COMPILE_FLAGS "-I${CMAKE_CURRENT_SOURCE_DIR}" )

  add_test(${NAME} ${NAME}test)
endmacro(ADD_CXXTEST)


## ============================================================================
## ============================================================================
## ============================================================================
# Build Macros

# copy_file ===================================================================
# copy a single file into build environment
macro(copy_file NAME)
  message(STATUS "Copying File: " ${NAME})
  file(INSTALL ${CMAKE_CURRENT_SOURCE_DIR}/${NAME}
    DESTINATION   ${CMAKE_CURRENT_BINARY_DIR} )
endmacro(copy_file NAME)

# copy_files ==================================================================
# copy all files matching a pattern into the build environment
macro(copy_files NAME)
  message(STATUS "Copying Files: " ${NAME})
  file(INSTALL ${CMAKE_CURRENT_SOURCE_DIR}/
    DESTINATION  ${CMAKE_CURRENT_BINARY_DIR}
    FILES_MATCHING PATTERN ${NAME} )
endmacro(copy_files NAME)

# add_graphlab_executable =====================================================
# copy all files matching a pattern into the build environment
macro(add_graphlab_executable NAME)
  add_executable(${NAME} ${ARGN})
  target_link_libraries(${NAME} graphlab)
  add_dependencies(${NAME} graphlab)
  requires_core_deps(${NAME})
endmacro(add_graphlab_executable)


macro(add_extension_executable NAME)
  add_executable(${NAME} extension_main.cpp ${ARGN})
  target_link_libraries(${NAME} graphlab graphlab_extension)
  add_dependencies(${NAME} graphlab graphlab_extension)
  requires_core_deps(${NAME})
endmacro(add_extension_executable)


add_custom_target(external_dependencies)
requires_core_deps(external_dependencies)

# add_jni_library =============================================================
# If jni was found then create a jni library.  Otherwise generate a warning
macro(add_jni_library NAME)
  # Only build if JNI was found
  if (JNI_REALLY_FOUND)
    include_directories(
      ${JAVA_INCLUDE_PATH}
      ${JAVA_INCLUDE_PATH}/linux
      )
    message(STATUS "Detected JNI library " ${NAME})
    add_library(${NAME} SHARED ${ARGN})
    target_link_libraries(${NAME}  graphlab_pic)
    target_link_libraries(${NAME} ${Boost_SHARED_LIBRARIES})
    # IF (APPLE)
    #         SET(CMAKE_SHARED_MODULE_CREATE_CXX_FLAGS "-dynamiclib -m64 ")
    #         SET_TARGET_PROPERTIES(${NAME} PROPERTIES SUFFIX .jnilib)
    # ENDIF (APPLE)
  elseif ()
    message(STATUS "Not building " ${NAME} " because JNI was not found")
  endif ()
endmacro(add_jni_library)
