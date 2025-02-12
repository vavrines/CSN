cmake_minimum_required( VERSION 3.16 )
project( SPM VERSION 0.1.0 LANGUAGES CXX )

### OPTIONS #####################################
option( BUILD_UNITY "enables unity build for faster compile times" ON )
#################################################

### COMPILER ####################################
set( CMAKE_CXX_STANDARD 17 )
set( CMAKE_CXX_STANDARD_REQUIRED ON )
set( KITRT_RELEASE_OPTIONS -march=native -w )
set( KITRT_RELWITHDEBINFO_OPTIONS -march=native -pg -no-pie )
set( KITRT_DEBUG_OPTIONS -Wall -Wextra -Wpedantic )
if( BUILD_UNITY AND NOT BUILD_CODE_COV )
    message( STATUS "Unity build enabled" )
    set( CMAKE_UNITY_BUILD ON )
    set( CMAKE_UNITY_BUILD_BATCH_SIZE 0 )
endif()
#################################################

### LIBRARIES ###################################
set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

find_package( OpenMP REQUIRED )

find_package( MPI REQUIRED )
include_directories( ${MPI_INCLUDE_PATH} )

find_package( LAPACK REQUIRED )
include_directories( ${LAPACK_INCLUDE_DIR} )

find_package( BLAS )

find_package( VTK COMPONENTS vtkIOGeometry vtkFiltersCore REQUIRED )

find_package( Python3 COMPONENTS Interpreter Development NumPy REQUIRED )
include_directories( ${Python3_INCLUDE_DIRS} ${Python3_NumPy_INCLUDE_DIRS} )
add_compile_definitions( KITRT_PYTHON_PATH="${CMAKE_SOURCE_DIR}/python" )

add_compile_definitions( BLAZE_USE_SHARED_MEMORY_PARALLELIZATION=0 )
message( STATUS "Blaze: Shared memory parallelization disabled" )
if( BLAS_FOUND )
    message( STATUS "Blaze: BLAS mode enabled" )
    add_compile_definitions( BLAZE_BLAS_MODE=1 )
    include_directories( ${BLAS_INCLUDE_DIR} )
else()
    message( STATUS "Blaze: BLAS mode disabled" )
    add_compile_definitions( BLAZE_BLAS_MODE=0 )
endif()
include( blaze-cache-config )
include_directories( ${CMAKE_SOURCE_DIR}/ext/blaze )

include_directories( ${CMAKE_SOURCE_DIR}/ext/cpptoml/include )
include_directories( ${CMAKE_SOURCE_DIR}/ext/spdlog/include )

set( CORE_LIBRARIES ${Python3_LIBRARIES} ${LAPACK_LIBRARIES} ${BLAS_LIBRARIES} ${MPI_LIBRARIES} ${VTK_LIBRARIES} OpenMP::OpenMP_CXX -lstdc++fs )
#################################################

### MISC ########################################
execute_process(
    COMMAND git rev-parse HEAD
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_HASH
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
add_compile_definitions( GIT_HASH="${GIT_HASH}" )
#################################################

### BUILD ################################
file( GLOB_RECURSE SRCS RELATIVE ${CMAKE_SOURCE_DIR} "src/*.cpp" "include/*.h" "include/*.hpp" )
#set( EXCLUDE_DIR "/gui/" )
#foreach( TMP_PATH ${SRCS} )
#    string( FIND ${TMP_PATH} ${EXCLUDE_DIR} EXCLUDE_DIR_FOUND )
#    if( NOT ${EXCLUDE_DIR_FOUND} EQUAL -1 )
#        list( REMOVE_ITEM SRCS ${TMP_PATH} )
#    endif()
#endforeach( TMP_PATH )
include_directories( ${CMAKE_SOURCE_DIR}/include )
add_executable( ${CMAKE_PROJECT_NAME} ${SRCS} ${EXT_SRCS} )
target_link_libraries( ${CMAKE_PROJECT_NAME} ${CORE_LIBRARIES} )
target_compile_options( ${CMAKE_PROJECT_NAME} PUBLIC "$<$<CONFIG:DEBUG>:${KITRT_DEBUG_OPTIONS}>" )
target_compile_options( ${CMAKE_PROJECT_NAME} PUBLIC "$<$<CONFIG:RELWITHDEBINFO>:${KITRT_RELWITHDEBINFO_OPTIONS}>" )
target_compile_options( ${CMAKE_PROJECT_NAME} PUBLIC "$<$<CONFIG:RELEASE>:${KITRT_RELEASE_OPTIONS}>" )
#################################################
