project(obs-ffmpeg)

if(MSVC)
	set(obs-ffmpeg_PLATFORM_DEPS
		w32-pthreads)
endif()

option(ENABLE_FFMPEG_LOGGING "Enables obs-ffmpeg logging" OFF)
option(ENABLE_NEW_MPEGTS_OUTPUT "Use native srt/rist mpegts output" ON)

find_package(FFmpeg REQUIRED
	COMPONENTS avcodec avfilter avdevice avutil swscale avformat swresample)
include_directories(${FFMPEG_INCLUDE_DIRS})

add_library(obs-ffmpeg MODULE)
add_library(OBS::ffmpeg ALIAS obs-ffmpeg)

add_subdirectory(ffmpeg-mux)
if(ENABLE_NEW_MPEGTS_OUTPUT)
  find_package(Librist)
  find_package(Libsrt)
endif()

configure_file(
	"${CMAKE_CURRENT_SOURCE_DIR}/obs-ffmpeg-config.h.in"
	"${CMAKE_CURRENT_BINARY_DIR}/obs-ffmpeg-config.h")
include_directories(${CMAKE_CURRENT_BINARY_DIR})

target_sources(
  obs-ffmpeg
  PRIVATE obs-ffmpeg.c
          obs-ffmpeg-video-encoders.c
          obs-ffmpeg-audio-encoders.c
          obs-ffmpeg-av1.c
          obs-ffmpeg-nvenc.c
          obs-ffmpeg-output.c
          obs-ffmpeg-mux.c
          obs-ffmpeg-mux.h
          obs-ffmpeg-hls-mux.c
          obs-ffmpeg-source.c
          obs-ffmpeg-compat.h
          obs-ffmpeg-formats.h
          ${CMAKE_BINARY_DIR}/config/obs-ffmpeg-config.h)

set(obs-ffmpeg_HEADERS
	obs-ffmpeg-compat.h
	obs-ffmpeg-formats.h
	obs-ffmpeg-mux.h)

target_link_libraries(
  obs-ffmpeg
  PRIVATE OBS::libobs
          OBS::media-playback
          OBS::opts-parser
          FFmpeg::avcodec
          FFmpeg::avfilter
          FFmpeg::avformat
          FFmpeg::avdevice
          FFmpeg::avutil
          FFmpeg::swscale
          FFmpeg::swresample)

if(ENABLE_NEW_MPEGTS_OUTPUT)
  target_sources(obs-ffmpeg PRIVATE obs-ffmpeg-mpegts.c obs-ffmpeg-srt.h
                                    obs-ffmpeg-rist.h obs-ffmpeg-url.h)

  target_link_libraries(obs-ffmpeg PRIVATE Librist::Librist Libsrt::Libsrt)
  if(OS_WINDOWS)
    target_link_libraries(obs-ffmpeg PRIVATE ws2_32.lib)
  endif()
  target_compile_definitions(obs-ffmpeg PRIVATE NEW_MPEGTS_OUTPUT)
endif()

if(UNIX AND NOT APPLE)
	list(APPEND obs-ffmpeg_SOURCES
		obs-ffmpeg-vaapi.c)
	LIST(APPEND obs-ffmpeg_PLATFORM_DEPS
		${LIBVA_LBRARIES}
		${LIBPCI_LIBRARIES})
endif()

set_target_properties(obs-ffmpeg PROPERTIES FOLDER "plugins/obs-ffmpeg" PREFIX
                                                                        "")
target_compile_options(
  obs-ffmpeg
  PRIVATE
    $<$<OR:$<C_COMPILER_ID:Clang>,$<C_COMPILER_ID:AppleClang>,$<C_COMPILER_ID:GNU>>:-Wno-switch>
)

if(OS_WINDOWS)
  if(MSVC)
    target_link_libraries(obs-ffmpeg PRIVATE OBS::w32-pthreads)
  endif()
endif()

if(WIN32)
	set(MODULE_DESCRIPTION "OBS FFmpeg module")
	configure_file(${CMAKE_SOURCE_DIR}/cmake/winrc/obs-module.rc.in obs-ffmpeg.rc)
	list(APPEND obs-ffmpeg_SOURCES
		jim-nvenc.c
		jim-nvenc-helpers.c
		obs-ffmpeg.rc)
	list(APPEND obs-ffmpeg_HEADERS
		jim-nvenc.h)
endif()

install_obs_plugin_with_data(obs-ffmpeg data)

add_subdirectory(ffmpeg-mux)
