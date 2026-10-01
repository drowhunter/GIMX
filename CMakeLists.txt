cmake_minimum_required(VERSION 3.20)

project(GIMX VERSION 0.0.0 LANGUAGES C CXX)

include(GNUInstallDirs)
include(CTest)
include_directories("${PROJECT_SOURCE_DIR}/shared")

option(GIMX_BUILD_UTILS "Build command-line utility programs" ON)
option(GIMX_USE_PDCURSES "Use PDCurses instead of ncurses for the core UI" OFF)

if(WIN32)
  enable_language(RC)
  add_compile_definitions(__USE_MINGW_ANSI_STDIO=0)
elseif(UNIX)
  add_compile_definitions(UHID)
endif()

find_package(CURL REQUIRED)
find_package(LibXml2 REQUIRED)
find_package(wxWidgets REQUIRED)
include(${wxWidgets_USE_FILE})

if(WIN32)
  find_path(LIBUSB_INCLUDE_DIR NAMES libusb.h PATH_SUFFIXES libusb-1.0 REQUIRED)
  find_library(LIBUSB_LIBRARY NAMES usb-1.0 libusb-1.0 REQUIRED)
  add_library(gimx_libusb UNKNOWN IMPORTED)
  set_target_properties(gimx_libusb PROPERTIES
    IMPORTED_LOCATION "${LIBUSB_LIBRARY}"
    INTERFACE_INCLUDE_DIRECTORIES "${LIBUSB_INCLUDE_DIR}")
  set(GIMX_LIBUSB_TARGET gimx_libusb)
  find_package(SDL2 REQUIRED CONFIG)
  if(TARGET SDL2::SDL2)
    set(GIMX_SDL_TARGET SDL2::SDL2)
  elseif(TARGET SDL2::SDL2-static)
    set(GIMX_SDL_TARGET SDL2::SDL2-static)
  else()
    set(GIMX_SDL_TARGET ${SDL2_LIBRARIES})
  endif()
else()
  find_package(PkgConfig REQUIRED)
  pkg_check_modules(LIBUSB REQUIRED IMPORTED_TARGET libusb-1.0)
  set(GIMX_LIBUSB_TARGET PkgConfig::LIBUSB)
  if(GIMX_USE_PDCURSES)
    find_library(PDCURSESW_LIBRARY NAMES pdcursesw REQUIRED)
  else()
    set(CURSES_NEED_WIDE TRUE)
    find_package(Curses REQUIRED)
  endif()
  find_package(X11 REQUIRED)
  find_library(BLUETOOTH_LIBRARY NAMES bluetooth REQUIRED)
  find_library(MHASH_LIBRARY NAMES mhash REQUIRED)
endif()

function(gimx_glob_sources output_var)
  set(collected_sources)
  foreach(pattern IN LISTS ARGN)
    file(GLOB matched_sources CONFIGURE_DEPENDS "${PROJECT_SOURCE_DIR}/${pattern}")
    list(APPEND collected_sources ${matched_sources})
  endforeach()
  set(${output_var} "${collected_sources}" PARENT_SCOPE)
endfunction()

function(gimx_add_shared target)
  add_library(${target} SHARED ${ARGN})
  set_target_properties(${target} PROPERTIES
    POSITION_INDEPENDENT_CODE ON
    PREFIX "lib"
  )
  target_include_directories(${target} PUBLIC
    "${PROJECT_SOURCE_DIR}/shared"
    "${PROJECT_SOURCE_DIR}/shared/gimxcommon/include"
    "${PROJECT_SOURCE_DIR}/shared/${target}"
    "${PROJECT_SOURCE_DIR}/shared/${target}/include"
  )
  target_compile_definitions(${target} PRIVATE GLOG_NAME=${target})
endfunction()

gimx_glob_sources(gimxlog_sources "shared/gimxlog/src/*.c")
gimx_add_shared(gimxlog ${gimxlog_sources})

if(WIN32)
  gimx_glob_sources(gimxtime_sources "shared/gimxtime/src/windows/*.c")
else()
  gimx_glob_sources(gimxtime_sources "shared/gimxtime/src/posix/*.c")
endif()
gimx_add_shared(gimxtime ${gimxtime_sources})

gimx_glob_sources(gimxcontroller_sources "shared/gimxcontroller/src/*.c")
gimx_add_shared(gimxcontroller ${gimxcontroller_sources})

gimx_glob_sources(gimxfile_sources "shared/gimxfile/src/*.cpp")
gimx_add_shared(gimxfile ${gimxfile_sources})
target_link_libraries(gimxfile PRIVATE gimxlog)
if(WIN32)
  target_link_libraries(gimxfile PRIVATE shell32 uuid ole32)
endif()

gimx_glob_sources(gimxdownloader_sources "shared/gimxdownloader/*.cpp")
gimx_add_shared(gimxdownloader ${gimxdownloader_sources})
target_link_libraries(gimxdownloader PRIVATE gimxfile CURL::libcurl)

gimx_glob_sources(gimxupdater_sources "shared/gimxupdater/*.cpp")
gimx_add_shared(gimxupdater ${gimxupdater_sources})
target_link_libraries(gimxupdater PRIVATE gimxfile gimxdownloader)

gimx_glob_sources(gimxconfigupdater_sources "shared/gimxconfigupdater/*.cpp")
gimx_add_shared(gimxconfigupdater ${gimxconfigupdater_sources})
target_link_libraries(gimxconfigupdater PRIVATE gimxfile gimxdownloader)

if(WIN32)
  gimx_glob_sources(gimxhid_sources
    "shared/gimxcommon/src/windows/async.c"
    "shared/gimxcommon/src/windows/gerror.c"
    "shared/gimxhid/src/windows/*.c")
else()
  gimx_glob_sources(gimxhid_sources "shared/gimxhid/src/libusb/*.c")
endif()
gimx_add_shared(gimxhid ${gimxhid_sources})
target_link_libraries(gimxhid PRIVATE gimxlog)
if(WIN32)
  target_link_libraries(gimxhid PRIVATE hid setupapi)
else()
  target_link_libraries(gimxhid PRIVATE ${GIMX_LIBUSB_TARGET})
endif()

if(WIN32)
  gimx_glob_sources(gimxpoll_sources
    "shared/gimxcommon/src/windows/gerror.c"
    "shared/gimxpoll/src/windows/*.c")
else()
  gimx_glob_sources(gimxpoll_sources "shared/gimxpoll/src/posix/*.c")
endif()
gimx_add_shared(gimxpoll ${gimxpoll_sources})
target_link_libraries(gimxpoll PRIVATE gimxlog)
if(WIN32)
  target_link_libraries(gimxpoll PRIVATE ws2_32)
endif()

if(WIN32)
  gimx_glob_sources(gimxprio_sources
    "shared/gimxcommon/src/windows/gerror.c"
    "shared/gimxprio/src/windows/*.c")
else()
  gimx_glob_sources(gimxprio_sources "shared/gimxprio/src/linux/*.c")
endif()
gimx_add_shared(gimxprio ${gimxprio_sources})
target_link_libraries(gimxprio PRIVATE gimxlog)

if(WIN32)
  gimx_glob_sources(gimxserial_sources
    "shared/gimxcommon/src/windows/async.c"
    "shared/gimxcommon/src/windows/gerror.c"
    "shared/gimxserial/src/windows/*.c")
else()
  gimx_glob_sources(gimxserial_sources
    "shared/gimxcommon/src/posix/async.c"
    "shared/gimxserial/src/posix/*.c")
endif()
gimx_add_shared(gimxserial ${gimxserial_sources})
target_link_libraries(gimxserial PRIVATE gimxlog)

if(WIN32)
  gimx_glob_sources(gimxtimer_sources
    "shared/gimxcommon/src/windows/gerror.c"
    "shared/gimxtimer/src/windows/*.c")
else()
  gimx_glob_sources(gimxtimer_sources "shared/gimxtimer/src/linux/*.c")
endif()
gimx_add_shared(gimxtimer ${gimxtimer_sources})
target_link_libraries(gimxtimer PRIVATE gimxlog gimxtime)

gimx_glob_sources(gimxgpp_sources "shared/gimxgpp/*.c")
gimx_add_shared(gimxgpp ${gimxgpp_sources})
target_link_libraries(gimxgpp PRIVATE gimxlog gimxhid gimxfile)

if(WIN32)
  gimx_glob_sources(gimxudp_sources
    "shared/gimxudp/src/posix/*.c"
    "shared/gimxudp/src/windows/*.c"
    "shared/gimxcommon/src/windows/gerror.c")
else()
  gimx_glob_sources(gimxudp_sources "shared/gimxudp/src/posix/*.c")
endif()
gimx_add_shared(gimxudp ${gimxudp_sources})
target_link_libraries(gimxudp PRIVATE gimxlog)
if(WIN32)
  target_link_libraries(gimxudp PRIVATE ws2_32)
endif()

if(UNIX)
  gimx_glob_sources(gimxuhid_sources "shared/gimxuhid/src/linux/*.c")
  gimx_add_shared(gimxuhid ${gimxuhid_sources})
  target_link_libraries(gimxuhid PRIVATE gimxlog)
endif()

gimx_glob_sources(gimxinput_sources
  "shared/gimxinput/src/*.c"
  "shared/gimxinput/src/hid/*.c")
if(WIN32)
  gimx_glob_sources(gimxinput_platform_sources
    "shared/gimxinput/src/windows/*.c"
    "shared/gimxinput/src/sdl/*.c")
  list(APPEND gimxinput_sources ${gimxinput_platform_sources})
else()
  gimx_glob_sources(gimxinput_platform_sources "shared/gimxinput/src/linux/*.c")
  list(APPEND gimxinput_sources ${gimxinput_platform_sources})
endif()
gimx_add_shared(gimxinput ${gimxinput_sources})
target_link_libraries(gimxinput PRIVATE gimxlog gimxhid gimxtime)
if(WIN32)
  target_link_libraries(gimxinput PRIVATE gimxpoll ${GIMX_SDL_TARGET} setupapi ws2_32)
else()
  target_link_libraries(gimxinput PRIVATE gimxuhid ${X11_Xi_LIB} ${X11_LIBRARIES})
endif()

gimx_glob_sources(gimxusb_sources "shared/gimxusb/src/*.c")
gimx_add_shared(gimxusb ${gimxusb_sources})
target_link_libraries(gimxusb PRIVATE gimxlog ${GIMX_LIBUSB_TARGET})
if(WIN32)
  target_link_libraries(gimxusb PRIVATE gimxtime gimxtimer gimxpoll)
endif()

gimx_glob_sources(gimxconfigeditor_sources "shared/gimxconfigeditor/src/*.cpp")
gimx_add_shared(gimxconfigeditor ${gimxconfigeditor_sources})
target_link_libraries(gimxconfigeditor PRIVATE
  LibXml2::LibXml2
  gimxhid gimxinput gimxtimer gimxpoll gimxcontroller)
if(UNIX)
  target_link_libraries(gimxconfigeditor PRIVATE gimxuhid)
endif()

gimx_glob_sources(gimx_core_sources
  "core/*.c"
  "core/connectors/*.c"
  "core/connectors/bluetooth/*.c"
  "core/connectors/bluetooth/btstack/*.c"
  "core/connectors/bluetooth/btstack/src/*.c"
  "core/connectors/gpp/*.c"
  "core/connectors/report2event/*.c"
  "core/haptic/*.c"
  "core/haptic/source/*.c"
  "core/haptic/sink/*.c"
  "core/haptic/common/*.c")
if(WIN32)
  gimx_glob_sources(gimx_core_platform_sources
    "core/system/windows/*.c"
    "core/connectors/windows/*.c"
    "core/connectors/windows/*/*.c"
    "core/connectors/bluetooth/windows/*.c")
  list(APPEND gimx_core_sources ${gimx_core_platform_sources})
  list(FILTER gimx_core_sources EXCLUDE REGEX "/core/connectors/btds4\\.c$")
else()
  gimx_glob_sources(gimx_core_platform_sources
    "core/system/linux/*.c"
    "core/connectors/linux/*/*.c"
    "core/connectors/bluetooth/linux/*.c")
  list(APPEND gimx_core_sources ${gimx_core_platform_sources})
endif()
add_executable(gimx ${gimx_core_sources})
target_include_directories(gimx PRIVATE
  "${PROJECT_SOURCE_DIR}/core"
  "${PROJECT_SOURCE_DIR}/core/connectors/bluetooth/btstack/include"
  "${PROJECT_SOURCE_DIR}/core/connectors/bluetooth/btstack/src")
target_link_libraries(gimx PRIVATE
  gimxfile gimxhid gimxgpp gimxcontroller gimxinput gimxpoll gimxprio
  gimxserial gimxtimer gimxusb gimxudp LibXml2::LibXml2)
if(WIN32)
  target_link_libraries(gimx PRIVATE ${GIMX_SDL_TARGET} ws2_32)
else()
  if(GIMX_USE_PDCURSES)
    target_compile_definitions(gimx PRIVATE PDCURSES)
    target_link_libraries(gimx PRIVATE ${PDCURSESW_LIBRARY})
  else()
    target_link_libraries(gimx PRIVATE ${CURSES_LIBRARIES})
    target_include_directories(gimx PRIVATE ${CURSES_INCLUDE_DIRS})
  endif()
  target_link_libraries(gimx PRIVATE m ${BLUETOOTH_LIBRARY} ${MHASH_LIBRARY})
endif()

function(gimx_add_wx_app target output_name source_pattern resource_file pch_file)
  gimx_glob_sources(app_sources "${source_pattern}")
  if(WIN32 AND NOT "${resource_file}" STREQUAL "")
    list(APPEND app_sources "${PROJECT_SOURCE_DIR}/${resource_file}")
  endif()
  add_executable(${target} ${app_sources})
  set_target_properties(${target} PROPERTIES OUTPUT_NAME "${output_name}")
  target_compile_definitions(${target} PRIVATE WX_PRECOMP)
  target_precompile_headers(${target} PRIVATE "${PROJECT_SOURCE_DIR}/${pch_file}")
  target_link_libraries(${target} PRIVATE ${wxWidgets_LIBRARIES})
endfunction()

gimx_add_wx_app(gimx_config "gimx-config" "config/*.cpp" "config/config.rc" "config/wx_pch.h")
target_link_libraries(gimx_config PRIVATE
  gimxconfigeditor gimxupdater gimxconfigupdater LibXml2::LibXml2)

gimx_add_wx_app(gimx_fpsconfig "gimx-fpsconfig" "fpsconfig/*.cpp" "fpsconfig/fpsconfig.rc" "fpsconfig/wx_pch.h")
target_link_libraries(gimx_fpsconfig PRIVATE
  gimxconfigeditor gimxupdater gimxconfigupdater LibXml2::LibXml2)

gimx_add_wx_app(gimx_launcher "gimx-launcher" "launcher/*.cpp" "launcher/launcher.rc" "launcher/wx_pch.h")
target_link_libraries(gimx_launcher PRIVATE
  gimxconfigeditor gimxupdater gimxconfigupdater gimxhid gimxgpp gimxserial)
if(WIN32)
  target_link_libraries(gimx_launcher PRIVATE gimxusb ws2_32)
endif()

gimx_add_wx_app(gimx_loader "gimx-loader" "loader/*.cpp" "" "loader/wx_pch.h")
target_link_libraries(gimx_loader PRIVATE gimxtime)

set(gimx_install_targets
  gimx gimx_config gimx_fpsconfig gimx_launcher gimx_loader)

if(GIMX_BUILD_UTILS)
  add_executable(ds4tool utils/ds4tool.c)
  add_executable(sixaddr utils/sixaddr.c)
  target_link_libraries(ds4tool PRIVATE gimxpoll gimxusb)
  target_link_libraries(sixaddr PRIVATE gimxpoll gimxusb)
  list(APPEND gimx_install_targets ds4tool sixaddr)
  if(UNIX)
    add_executable(bdaddr utils/linux/bdaddr.c utils/linux/oui.c)
    add_executable(hcirevision utils/linux/hcirevision.c)
    target_link_libraries(bdaddr PRIVATE gimxpoll gimxusb ${BLUETOOTH_LIBRARY})
    target_link_libraries(hcirevision PRIVATE gimxpoll gimxusb ${BLUETOOTH_LIBRARY})
    list(APPEND gimx_install_targets bdaddr hcirevision)
  endif()
endif()

install(TARGETS ${gimx_install_targets}
  RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
  LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
  ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}")
install(TARGETS
  gimxlog gimxtime gimxcontroller gimxfile gimxdownloader gimxupdater
  gimxconfigupdater gimxhid gimxinput gimxpoll gimxprio gimxserial gimxtimer
  gimxusb gimxgpp gimxudp gimxconfigeditor
  RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
  LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
  ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}")
if(UNIX)
  install(TARGETS gimxuhid
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}")
endif()

install(DIRECTORY "${PROJECT_SOURCE_DIR}/loader/firmware/"
  DESTINATION "${CMAKE_INSTALL_DATADIR}/gimx/firmware"
  FILES_MATCHING PATTERN "*.hex")
if(EXISTS "${PROJECT_SOURCE_DIR}/shared/gimxinput/src/windows/gamecontrollerdb.txt")
  install(FILES "${PROJECT_SOURCE_DIR}/shared/gimxinput/src/windows/gamecontrollerdb.txt"
    DESTINATION "${CMAKE_INSTALL_DATADIR}/gimx")
endif()
foreach(app config fpsconfig launcher)
  if(EXISTS "${PROJECT_SOURCE_DIR}/${app}/gimx-${app}.desktop")
    install(FILES "${PROJECT_SOURCE_DIR}/${app}/gimx-${app}.desktop"
      DESTINATION "${CMAKE_INSTALL_DATADIR}/applications")
  endif()
  if(EXISTS "${PROJECT_SOURCE_DIR}/${app}/gimx-${app}.png")
    install(FILES "${PROJECT_SOURCE_DIR}/${app}/gimx-${app}.png"
      DESTINATION "${CMAKE_INSTALL_DATADIR}/pixmaps")
  endif()
endforeach()

find_program(MSGFMT_EXECUTABLE msgfmt)
if(MSGFMT_EXECUTABLE)
  file(GLOB translation_files CONFIGURE_DEPENDS "${PROJECT_SOURCE_DIR}/po/*.po")
  set(compiled_translations)
  foreach(translation IN LISTS translation_files)
    get_filename_component(language "${translation}" NAME_WE)
    set(compiled_translation "${PROJECT_BINARY_DIR}/locale/${language}/LC_MESSAGES/gimx.mo")
    add_custom_command(OUTPUT "${compiled_translation}"
      COMMAND "${CMAKE_COMMAND}" -E make_directory "${PROJECT_BINARY_DIR}/locale/${language}/LC_MESSAGES"
      COMMAND "${MSGFMT_EXECUTABLE}" -o "${compiled_translation}" "${translation}"
      DEPENDS "${translation}"
      VERBATIM)
    list(APPEND compiled_translations "${compiled_translation}")
    install(FILES "${compiled_translation}"
      DESTINATION "${CMAKE_INSTALL_DATADIR}/locale/${language}/LC_MESSAGES")
  endforeach()
  add_custom_target(translations ALL DEPENDS ${compiled_translations})
endif()