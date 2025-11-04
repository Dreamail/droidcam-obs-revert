cmake_minimum_required(VERSION 3.28...3.30)

include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/common/bootstrap.cmake" NO_POLICY_SCOPE)

project(${_name} VERSION ${_version})

option(ENABLE_FRONTEND_API "Use obs-frontend-api for UI functionality" OFF)
option(ENABLE_QT "Use Qt functionality" OFF)

include(compilerconfig)
include(defaults)
include(helpers)

add_library(${CMAKE_PROJECT_NAME} MODULE)

find_package(libobs REQUIRED)
target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE OBS::libobs)

if(ENABLE_FRONTEND_API)
  find_package(obs-frontend-api REQUIRED)
  target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE OBS::obs-frontend-api)
endif()

if(ENABLE_QT)
  find_package(Qt6 COMPONENTS Widgets Core)
  target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE Qt6::Core Qt6::Widgets)
  target_compile_options(
    ${CMAKE_PROJECT_NAME}
    PRIVATE $<$<C_COMPILER_ID:Clang,AppleClang>:-Wno-quoted-include-in-framework-header -Wno-comma>
  )
  set_target_properties(
    ${CMAKE_PROJECT_NAME}
    PROPERTIES AUTOMOC ON AUTOUIC ON AUTORCC ON
  )
endif()

find_package(unofficial-libimobiledevice CONFIG REQUIRED)
target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE unofficial::libimobiledevice::libimobiledevice)

find_package(unofficial-libusbmuxd CONFIG REQUIRED)
target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE unofficial::libusbmuxd::libusbmuxd)

find_package(FFMPEG REQUIRED)
target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE ${FFMPEG_INCLUDE_DIRS})
target_link_directories(${CMAKE_PROJECT_NAME} PRIVATE ${FFMPEG_LIBRARY_DIRS})
target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE ${FFMPEG_LIBRARIES})

find_package(libjpeg-turbo CONFIG REQUIRED)
target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE $<IF:$<TARGET_EXISTS:libjpeg-turbo::turbojpeg>,libjpeg-turbo::turbojpeg,libjpeg-turbo::turbojpeg-static>)

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE ./src)

target_sources(${CMAKE_PROJECT_NAME} PRIVATE
    src/device_discovery.cc
    src/ffmpeg_decode.cc
    src/mdns_discovery.cc
    src/mjpeg_decode.cc
    src/net.cc
    src/plugin.cc
    src/proxy.cc
    src/source.cc
)

if(WIN32)
  target_sources(${CMAKE_PROJECT_NAME} PRIVATE
      src/sys/win/cmd.cc
      src/sys/win/util.cc
  )
elseif(UNIX)
  target_sources(${CMAKE_PROJECT_NAME} PRIVATE
      src/sys/unix/cmd.cc
      src/sys/unix/util.cc
  )
endif()

if (OS_LINUX)
  # for ffmpeg PIC compatibility, see: https://github.com/microsoft/vcpkg/issues/17292#issuecomment-935943256
  set(CMAKE_SHARED_LINKER_FLAGS "-Wl,-Bsymbolic")
endif()

set_target_properties_plugin(${CMAKE_PROJECT_NAME} PROPERTIES OUTPUT_NAME ${_name})
