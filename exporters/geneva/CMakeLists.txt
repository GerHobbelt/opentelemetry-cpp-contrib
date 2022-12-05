cmake_minimum_required(VERSION 3.12)

# MAIN_PROJECT CHECK determine if fluentd exporter is built as a subproject
# (using add_subdirectory) or if it is the main project
#
set(MAIN_PROJECT OFF)
set(MAIN_PROJECT OFF)
if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
  project(opentelemetry-geneva-metrics)
  set(MAIN_PROJECT ON)
endif()

if(NOT WIN32)
  find_package(CURL REQUIRED)
endif()
add_definitions(-DHAVE_CONSOLE_LOG)
if(MAIN_PROJECT)
  find_package(opentelemetry-cpp REQUIRED)
endif()

include_directories(include)

# create geneva metrics exporter
if(WIN32)
  add_library(
    opentelemetry_exporter_geneva_metrics
    src/exporter.cc src/etw_data_transport.cc
    src/unix_domain_socket_data_transport.cc)
else()
  add_library(opentelemetry_exporter_geneva_metrics
              src/exporter.cc src/unix_domain_socket_data_transport.cc)
endif()

target_include_directories(opentelemetry_exporter_geneva_metrics
                           PRIVATE ${OPENTELEMETRY_CPP_INCLUDE_DIRS})
target_link_libraries(opentelemetry_exporter_geneva_metrics
                      PUBLIC ${OPENTELEMETRY_CPP_LIBRARIES})
set_target_properties(opentelemetry_exporter_geneva_metrics
                      PROPERTIES EXPORT_NAME metrics)

if(BUILD_TESTING)
  if(EXISTS ${CMAKE_BINARY_DIR}/lib/libgtest.a)
    # Prefer GTest from build tree. GTest is not always working with
    # CMAKE_PREFIX_PATH
    set(GTEST_INCLUDE_DIRS
        ${CMAKE_CURRENT_SOURCE_DIR}/third_party/googletest/googletest/include
        ${CMAKE_CURRENT_SOURCE_DIR}/third_party/googletest/googlemock/include)
    set(GTEST_BOTH_LIBRARIES
        ${CMAKE_BINARY_DIR}/lib/libgtest.a
        ${CMAKE_BINARY_DIR}/lib/libgtest_main.a
        ${CMAKE_BINARY_DIR}/lib/libgmock.a)
  else()
    find_package(GTest REQUIRED)
  endif()
  include_directories(SYSTEM ${GTEST_INCLUDE_DIRS})
  include_directories(SYSTEM test/decoder)
  enable_testing()
  include(GoogleTest)
  # build geneva metrics tests
  add_compile_definitions(KS_STR_ENCODING_NONE HAVE_UNIX_DOMAIN)
  add_executable(
    geneva_metrics_exporter_test
    test/metrics_exporter_test.cc test/decoder/ifx_metrics_bin.cpp
    test/decoder/kaitai/kaitaistream.cpp)
  target_link_libraries(
    geneva_metrics_exporter_test ${GTEST_BOTH_LIBRARIES}
    ${CMAKE_THREAD_LIBS_INIT} opentelemetry_exporter_geneva_metrics)

  gtest_add_tests(
    TARGET geneva_metrics_exporter_test
    TEST_PREFIX exporter.
    TEST_LIST geneva_metrics_exporter_test)
endif()

install(
  TARGETS opentelemetry_exporter_geneva_metrics
  EXPORT "${PROJECT_NAME}-target"
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

install(
  DIRECTORY include/opentelemetry/exporters/geneva
  DESTINATION include/opentelemetry/exporters
  FILES_MATCHING
  PATTERN "*.h")

if(BUILD_EXAMPLE)
  add_executable(example_metrics example/example_metrics.cc
                                 example/foo_library.cc)
  target_link_libraries(example_metrics opentelemetry_exporter_geneva_metrics)
endif()