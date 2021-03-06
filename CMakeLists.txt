cmake_minimum_required(VERSION 3.1)

set (CMAKE_CXX_STANDARD 11)
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

option(BENCHMARK_ENABLE_TESTING "Enable testing of the benchmark library." OFF)
if(NOT WIN32)
  add_definitions(-Wno-error)
endif()
add_subdirectory(benchmark EXCLUDE_FROM_ALL)

include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-march=native" COMPILER_SUPPORTS_MARCH_NATIVE)

function(fetch_eigen VERSION NAME)
  if(NOT EXISTS ${CMAKE_CURRENT_BINARY_DIR}/eigen-${NAME})
    message("Downloading Eigen ${VERSION} ...")
    file(DOWNLOAD https://bitbucket.org/eigen/eigen/get/${VERSION}.tar.bz2 ${CMAKE_CURRENT_BINARY_DIR}/${VERSION}.tar.bz2)
    execute_process(
      COMMAND ${CMAKE_COMMAND} -E tar xjf ${CMAKE_CURRENT_BINARY_DIR}/${VERSION}.tar.bz2
      WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
    file(GLOB EIGEN_DIR "eigen-eigen-*")
    file(RENAME ${EIGEN_DIR} ${CMAKE_CURRENT_BINARY_DIR}/eigen-${NAME})
    file(REMOVE ${CMAKE_CURRENT_BINARY_DIR}/${VERSION}.tar.bz2)
  endif()
endfunction()

function(build_eigen_bench_impl VERSION NAME NATIVE)
  fetch_eigen(${VERSION} ${NAME})
  set(EIGEN_DIR ${CMAKE_CURRENT_BINARY_DIR}/eigen-${NAME})
  if(NATIVE)
    set(NAME ${NAME}_native)
  endif()
  set(BENCH_NAME bench_${NAME})
  configure_file(bench.in.cpp ${CMAKE_CURRENT_BINARY_DIR}/bench_${NAME}.cpp)
  add_executable(bench_${NAME} ${CMAKE_CURRENT_BINARY_DIR}/bench_${NAME}.cpp)
  set_target_properties(bench_${NAME} PROPERTIES FOLDER "Executables")
  if(NATIVE)
    target_compile_options(bench_${NAME} PRIVATE "-march=native")
  endif()
  target_include_directories(bench_${NAME} SYSTEM PRIVATE ${EIGEN_DIR})
  target_link_libraries(bench_${NAME} benchmark)
  if(NOT WIN32)
    target_link_libraries(bench_${NAME} pthread)
  endif()
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/FORCE_BENCH.rule
    COMMAND bench_${NAME}
    COMMENT "Running benchmarks"
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR} ${APPEND_BENCHMARK})
endfunction()

function(build_eigen_bench VERSION NAME)
  build_eigen_bench_impl(${VERSION} ${NAME} 0)
  set(APPEND_BENCHMARK APPEND PARENT_SCOPE)
  set(APPEND_BENCHMARK APPEND)
  if(COMPILER_SUPPORTS_MARCH_NATIVE)
    build_eigen_bench_impl(${VERSION} ${NAME} 1)
  endif()
endfunction()

add_custom_target(run_benchmark DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/FORCE_BENCH.rule)

build_eigen_bench(3.2.8 3_2_8)
build_eigen_bench(3.2.10 3_2_10)
build_eigen_bench(3.3-beta2 3_3_beta)
build_eigen_bench(3.3.4 3_3_4)
build_eigen_bench(3.3 trunk)

