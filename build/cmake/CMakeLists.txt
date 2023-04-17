if ( NOT DEFINED CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE Release CACHE STRING "Build type" )
endif ()

project (glew C)

cmake_minimum_required (VERSION 2.8.12)

include(GNUInstallDirs)

if(POLICY CMP0003)
  cmake_policy (SET CMP0003 NEW)
endif()

if(POLICY CMP0042)
  cmake_policy (SET CMP0042 NEW)
endif()


option (BUILD_UTILS "utilities" ON)
option (GLEW_REGAL "Regal mode" OFF)
option (GLEW_OSMESA "OSMesa mode" OFF)
if (APPLE)
    option (BUILD_FRAMEWORK "Build Framework bundle for OSX" OFF)
endif ()

set (GLEW_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../..)

# get version from config/version
file (STRINGS ${GLEW_DIR}/config/version  _VERSION_MAJOR_STRING REGEX "GLEW_MAJOR[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MAJOR[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_MAJOR ${_VERSION_MAJOR_STRING})
file (STRINGS ${GLEW_DIR}/config/version  _VERSION_MINOR_STRING REGEX "GLEW_MINOR[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MINOR[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_MINOR ${_VERSION_MINOR_STRING})
file (STRINGS ${GLEW_DIR}/config/version  _VERSION_PATCH_STRING REGEX "GLEW_MICRO[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MICRO[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_PATCH ${_VERSION_PATCH_STRING})
set (GLEW_VERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH})

set (CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set (CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set (CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

if (WIN32)
	find_package (OpenGL REQUIRED)
endif()

# X11 required except for Windows and Apple OSX platforms
if (NOT WIN32 AND NOT APPLE)
  find_package (X11)
endif()

set (GLEW_LIB_NAME glew)

set (GLEW_LIBRARIES ${OPENGL_LIBRARIES} ${X11_LIBRARIES})

add_definitions (-DGLEW_NO_GLU)

#### Regal mode ####

if (GLEW_REGAL)
  if (WIN32)
    set (REGAL_LIB_NAME regal32)
  else ()
    set (REGAL_LIB_NAME Regal)
  endif ()
  add_definitions (-DGLEW_REGAL)
  set (GLEW_LIBRARIES ${REGAL_LIB_NAME})
endif ()

#### OSMesa mode ####

if (GLEW_OSMESA)
  if (WIN32)
    set (OSMESA_LIB_NAME osmesa)
  else ()
    set (OSMESA_LIB_NAME OSMesa)
  endif ()
  add_definitions (-DGLEW_OSMESA)
  set (GLEW_LIBRARIES ${OSMESA_LIB_NAME} ${OPENGL_LIBRARIES})
  set (X11_LIBRARIES)
endif ()

#### EGL ####

if (GLEW_EGL AND UNIX)
  add_definitions (-DGLEW_EGL)
  if (OpenGL::EGL)
    message (FATAL_ERROR "EGL library set but not found.")
  endif()
  set (GLEW_LIBRARIES ${OPENGL_LIBRARIES} ${OPENGL_egl_LIBRARY})
endif ()

#### GLEW ####

include_directories (${GLEW_DIR}/include ${X11_INCLUDE_DIR})

set (GLEW_PUBLIC_HEADERS_FILES ${GLEW_DIR}/include/GL/wglew.h ${GLEW_DIR}/include/GL/glew.h ${GLEW_DIR}/include/GL/glxew.h)
set (GLEW_FILES ${GLEW_DIR}/src/glew.c)

if (WIN32)
  list (APPEND GLEW_FILES ${GLEW_DIR}/build/glew.rc)
endif ()

if(BUILD_SHARED_LIBS)
  add_library (GLEW SHARED ${GLEW_PUBLIC_HEADERS_FILES} ${GLEW_FILES})
  set_target_properties (GLEW PROPERTIES COMPILE_DEFINITIONS "GLEW_BUILD" OUTPUT_NAME "${GLEW_LIB_NAME}")
else(BUILD_SHARED_LIBS)
  add_library (GLEW STATIC ${GLEW_PUBLIC_HEADERS_FILES} ${GLEW_FILES})
  set_target_properties (GLEW PROPERTIES COMPILE_DEFINITIONS "GLEW_STATIC" OUTPUT_NAME "${GLEW_LIB_NAME}")
endif(BUILD_SHARED_LIBS)

if (MSVC)
  # add options from visual studio project
  if(BUILD_SHARED_LIBS)
    target_compile_definitions (GLEW PRIVATE "GLEW_BUILD;VC_EXTRALEAN")
  else(BUILD_SHARED_LIBS)
    target_compile_definitions (GLEW PRIVATE "GLEW_STATIC;VC_EXTRALEAN")
  endif(BUILD_SHARED_LIBS)
  target_link_libraries (GLEW LINK_PRIVATE -BASE:0x62AA0000)
  # kill security checks which are dependent on stdlib
  target_compile_options (GLEW PRIVATE -GS-)
  string(REGEX REPLACE "/RTC(su|[1su])" "" CMAKE_C_FLAGS_DEBUG ${CMAKE_C_FLAGS_DEBUG})
elseif (WIN32 AND ((CMAKE_C_COMPILER_ID MATCHES "GNU") OR (CMAKE_C_COMPILER_ID MATCHES "Clang")))
  # remove stdlib dependency on windows with GCC and Clang (for similar reasons
  # as to MSVC - to allow it to be used with any Windows compiler)
  target_compile_options (GLEW PRIVATE -fno-builtin -fno-stack-protector)
  target_link_libraries (GLEW LINK_PRIVATE -nostdlib)
endif ()

if (BUILD_FRAMEWORK)
  set_target_properties(GLEW PROPERTIES
    FRAMEWORK TRUE
    FRAMEWORK_VERSION ${GLEW_VERSION}
    MACOSX_FRAMEWORK_IDENTIFIER net.sourceforge.glew
    MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${GLEW_VERSION}
    MACOSX_FRAMEWORK_BUNDLE_VERSION ${GLEW_VERSION}
    XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
    PUBLIC_HEADER "${GLEW_PUBLIC_HEADERS_FILES}"
    OUTPUT_NAME GLEW
  )
endif()

target_link_libraries (GLEW LINK_PUBLIC ${GLEW_LIBRARIES})

if(CMAKE_VERSION VERSION_LESS 2.8.12)
  set(MAYBE_EXPORT "")
else()
  if(NOT BUILD_SHARED_LIBS)
    target_compile_definitions(GLEW INTERFACE "GLEW_STATIC")
  endif()
  target_include_directories(GLEW PUBLIC $<INSTALL_INTERFACE:include>)
  set(MAYBE_EXPORT EXPORT glew-targets)
endif()

set(targets_to_install "")
if(NOT DEFINED BUILD_SHARED_LIBS OR BUILD_SHARED_LIBS)
  list(APPEND targets_to_install GLEW)
endif()

if(NOT DEFINED BUILD_SHARED_LIBS OR NOT BUILD_SHARED_LIBS)
  list(APPEND targets_to_install GLEW)
endif()

install ( TARGETS ${targets_to_install}
          ${MAYBE_EXPORT}
          RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
          LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
          ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
          FRAMEWORK DESTINATION ${CMAKE_INSTALL_PREFIX}
)

if (BUILD_UTILS)
  set (GLEWINFO_SRC_FILES ${GLEW_DIR}/src/glewinfo.c)
  if (WIN32)
    list (APPEND GLEWINFO_SRC_FILES ${GLEW_DIR}/build/glewinfo.rc)
  endif ()
  add_executable (glewinfo ${GLEWINFO_SRC_FILES})
  if(NOT DEFINED BUILD_SHARED_LIBS OR BUILD_SHARED_LIBS)
     target_link_libraries (glewinfo GLEW)
  else()
     target_link_libraries (glewinfo GLEW)
  endif()
  if (NOT WIN32)
    target_link_libraries(glewinfo ${X11_LIBRARIES})
  endif ()

  set (VISUALINFO_SRC_FILES ${GLEW_DIR}/src/visualinfo.c)
  if (WIN32)
    list (APPEND VISUALINFO_SRC_FILES ${GLEW_DIR}/build/visualinfo.rc)
  endif ()
  add_executable (visualinfo ${VISUALINFO_SRC_FILES})
  if(NOT DEFINED BUILD_SHARED_LIBS OR BUILD_SHARED_LIBS)
     target_link_libraries (visualinfo GLEW)
  else()
     target_link_libraries (visualinfo GLEW)
  endif()
  if (NOT WIN32)
    target_link_libraries(visualinfo ${X11_LIBRARIES})
  endif ()

  install ( TARGETS glewinfo visualinfo
            DESTINATION ${CMAKE_INSTALL_BINDIR})
endif ()

set (prefix ${CMAKE_INSTALL_PREFIX})
set (exec_prefix ${CMAKE_INSTALL_PREFIX})
set (libdir ${CMAKE_INSTALL_FULL_LIBDIR})
set (includedir ${CMAKE_INSTALL_FULL_INCLUDEDIR})
set (version ${GLEW_VERSION})
set (libname ${GLEW_LIB_NAME})
set (cflags)
set (requireslib glu)

#  Mac OSX has no glu.pc unless optional X11/GLX is installed
if (APPLE)
  set (requireslib)
endif ()

install (FILES
    ${GLEW_DIR}/include/GL/wglew.h
    ${GLEW_DIR}/include/GL/glew.h
    ${GLEW_DIR}/include/GL/glxew.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/GL)

if(MAYBE_EXPORT)
  install(EXPORT glew-targets DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/glew
    NAMESPACE GLEW::)
  install(FILES
      ${CMAKE_CURRENT_SOURCE_DIR}/glew-config.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/glew)
endif()

if(NOT TARGET uninstall)
  configure_file(
      ${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in
      ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
      IMMEDIATE @ONLY)

  add_custom_target(uninstall
      COMMAND ${CMAKE_COMMAND} -P
      ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)
endif()
