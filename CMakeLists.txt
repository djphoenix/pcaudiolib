cmake_minimum_required(VERSION 3.15)

project(pcaudio VERSION 1.2)

if (BUILD_SHARED_LIBS)
  set(PCAUDIO_LIBRARY_MODE_DEF SHARED)
else()
  set(PCAUDIO_LIBRARY_MODE_DEF STATIC)
endif()
set(PCAUDIO_LIBRARY_MODE "${PCAUDIO_LIBRARY_MODE_DEF}" CACHE STRING "PCAudio Library mode (static/shared/object)")

add_library(pcaudio ${PCAUDIO_LIBRARY_MODE} src/audio.c)

# PulseAudio
find_library(PULSE pulse)
find_library(PULSE_SIMPLE pulse-simple)
find_path(PULSE_INC pulse/pulseaudio.h)
if(PULSE AND PULSE_SIMPLE AND PULSE_INC)
  set(HAVE_PULSE_SIMPLE_H ON)
  target_include_directories(pcaudio PRIVATE ${PULSE_INC})
  target_link_libraries(pcaudio PRIVATE ${PULSE} ${PULSE_SIMPLE})
endif()

# Alsa
find_library(ASOUND asound)
find_path(ASOUND_INC alsa/asoundlib.h)
if(ASOUND AND ASOUND_INC)
  set(HAVE_ALSA_ASOUNDLIB_H ON)
  target_include_directories(pcaudio PRIVATE ${ASOUND_INC})
  target_link_libraries(pcaudio PRIVATE ${ASOUND})
endif()

# OSS
find_path(OSS_INC sys/soundcard.h)
if(OSS_INC)
  set(HAVE_SYS_SOUNDCARD_H ON)
  target_include_directories(pcaudio PRIVATE ${OSS_INC})
endif()

# TODO: CoreAudio

configure_file(config.cmake.h.in config.h)

target_include_directories(pcaudio PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_include_directories(pcaudio PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/src/include)
set_target_properties(pcaudio PROPERTIES PUBLIC_HEADER "src/include/pcaudiolib/audio.h")

if (APPLE)
  target_sources(pcaudio PRIVATE
    src/coreaudio.c
    src/TPCircularBuffer/TPCircularBuffer.c
    src/TPCircularBuffer/TPCircularBuffer+AudioBufferList.c
  )
elseif (LINUX)
  target_sources(pcaudio PRIVATE
    src/alsa.c
    src/qsa.c
    src/oss.c
    src/pulseaudio.c
  )
elseif (WIN32)
  set_target_properties(pcaudio PROPERTIES
    MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>"
  )
  if (PCAUDIO_LIBRARY_MODE STREQUAL "SHARED")
    target_compile_definitions(pcaudio PRIVATE "PCAUDIOLIB_EXPORT=__declspec(dllexport)")
    target_compile_definitions(pcaudio INTERFACE "PCAUDIOLIB_EXPORT=__declspec(dllimport)")
  endif()
  target_sources(pcaudio PRIVATE
    src/windows.c
    src/xaudio2.cpp
  )
endif()

message(STATUS "Build mode:         ${PCAUDIO_LIBRARY_MODE}")
message(STATUS "PulseAudio support: ${PULSE} ${PULSE_INC}")
message(STATUS "ALSA support:       ${ASOUND} ${ASOUND_INC}")
message(STATUS "OSS support:        ${OSS_INC}")

include(CPack)
include(GNUInstallDirs)
install(
  TARGETS pcaudio
  PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/pcaudiolib/
)
