cmake_minimum_required(VERSION 3.12)

project(opentelemetry-nginx)

option(WITH_ABSEIL "Use abseil" OFF)

find_package(opentelemetry-cpp CONFIG REQUIRED)
find_package(nlohmann_json)
find_package(Threads REQUIRED)
find_package(Protobuf REQUIRED)
find_package(CURL REQUIRED)

if (WITH_ABSEIL)
  find_package(absl REQUIRED)
endif()

include(${CMAKE_CURRENT_SOURCE_DIR}/nginx.cmake)

add_library(otel_ngx_module SHARED
  src/nginx_config.cpp
  src/trace_context.cpp
  src/otel_ngx_module.cpp
  src/otel_ngx_module_modules.c
  src/propagate.cpp
  src/script.cpp
)

target_compile_options(otel_ngx_module
  PRIVATE -Wall -Wextra
)

install(TARGETS otel_ngx_module DESTINATION ".")

# Omit the lib prefix to be more in line with nginx module naming
set_target_properties(otel_ngx_module PROPERTIES
  PREFIX ""
)

add_dependencies(otel_ngx_module project_nginx)

target_include_directories(otel_ngx_module
  PRIVATE
  ${NGINX_INCLUDE_DIRS}
  ${OPENTELEMETRY_CPP_INCLUDE_DIRS}
)
target_link_libraries(otel_ngx_module
  PRIVATE
  ${OPENTELEMETRY_CPP_LIBRARIES}
  CURL::libcurl
)
