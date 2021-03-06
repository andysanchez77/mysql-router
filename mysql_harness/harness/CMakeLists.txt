# Copyright (c) 2015, 2016, Oracle and/or its affiliates. All rights reserved.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; version 2 of the License.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA  02110-1301  USA

find_package(Threads REQUIRED)

include(GenerateExportHeader)

set(harness_source
  src/loader.cc src/utilities.cc src/config_parser.cc src/designator.cc
  src/filesystem.cc
  src/arg_handler.cc
  src/networking/ip_address.cc
  src/networking/ipv4_address.cc
  src/networking/ipv6_address.cc
  src/networking/resolver.cc)
if(WIN32)
  list(APPEND harness_source
    src/filesystem-windows.cc src/utilities-windows.cc src/loader-windows.cc)
else()
  list(APPEND harness_source
    src/filesystem-posix.cc src/utilities-posix.cc src/loader-posix.cc)
endif()

file(GLOB harness_headers include/*.h)

include_directories(
  ${CMAKE_CURRENT_SOURCE_DIR}/include
  ${CMAKE_CURRENT_SOURCE_DIR}/src
  ${CMAKE_BINARY_DIR}/include)

if(WIN32)
  set(WINSOCK_LIBRARIES Ws2_32.lib)
endif()
set(common_libraries ${CMAKE_DL_LIBS} ${CMAKE_THREAD_LIBS_INIT} ${SHLWAPI_LIBRARIES} ${WINSOCK_LIBRARIES})

configure_file(plugin.h.in ${CMAKE_BINARY_DIR}/${INSTALL_INCLUDE_DIR}/plugin.h
  ESCAPE_QUOTES @ONLY)

# create harness library - static version
add_library(harness-archive STATIC ${harness_source})
target_link_libraries(harness-archive ${common_libraries})
target_include_directories(harness-archive PUBLIC include)
if(WIN32)
  set_target_properties(harness-archive PROPERTIES
		COMPILE_FLAGS -DHARNESS_STATIC_DEFINE)
endif()
if(THREADS_HAVE_PTHREAD_ARG)
  target_compile_options(PUBLIC harness-archive "-pthread")
endif()

# create harness library - dynamic version
add_library(harness-library SHARED ${harness_source})
target_link_libraries(harness-library ${common_libraries})
target_include_directories(harness-library PUBLIC include)

generate_export_header(harness-library
	BASE_NAME HARNESS
	EXPORT_FILE_NAME ${CMAKE_BINARY_DIR}/include/harness_export.h)

if(${CMAKE_SYSTEM_NAME} STREQUAL "SunOS")
  target_link_libraries(harness-library -lnsl -lsocket)
  target_link_libraries(harness-archive -lnsl -lsocket)
endif()

# This is the harness containing a basic main function.
if(ENABLE_HARNESS_PROGRAM)
  add_executable(harness-program src/main.cc)
  set_target_properties(harness-program PROPERTIES
    OUTPUT_NAME harness)
  if(NOT WIN32)
    target_link_libraries(harness-program
      ${common_libraries} harness-library)
  else()
    target_link_libraries(harness-program
      ${common_libraries} harness-archive)
    target_compile_definitions(harness-program PRIVATE -DHARNESS_STATIC_DEFINE=1)
  endif()
  install(TARGETS harness-program RUNTIME DESTINATION bin)
endif()

if(NOT WIN32)
  set_target_properties(harness-archive harness-library PROPERTIES
    OUTPUT_NAME "mysqlharness"
    PREFIX "lib"
    SOVERSION 1)
  set_target_properties(harness-library PROPERTIES
      LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/stage/lib)
else()
  set_target_properties(harness-archive PROPERTIES
    OUTPUT_NAME "mysqlharness_a"
    PREFIX "lib"
    SOVERSION 1)

  # Fix install path in stage dir, so that files get to
  # stage/${configuration_type}/{bin,etc,lib} instead of
  # stage/{bin,etc,lib}/${configuration_type}
  foreach(config_ ${CMAKE_CONFIGURATION_TYPES})
    set_target_properties(harness-library PROPERTIES
      RUNTIME_OUTPUT_DIRECTORY_${config_} ${CMAKE_BINARY_DIR}/stage/${config_}/bin) # <-- [SEARCH TAG] RUNTIME_OUTPUT_DIRECTORY
  endforeach()

  get_property(harness_library_location TARGET harness-library PROPERTY LOCATION)
  foreach(conf ${CMAKE_CONFIGURATION_TYPES})
    add_custom_command(TARGET harness-library
          COMMAND ${CMAKE_COMMAND} -E copy "${harness_library_location}" ${STAGE_DIR}/${conf}/lib/
          COMMENT "Copying harness library from: ${harness_library_location}"
    )
    file(MAKE_DIRECTORY "${STAGE_DIR}/${conf}/lib/")
  endforeach()
endif()

# copy harness headers
file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/include/ DESTINATION ${CMAKE_BINARY_DIR}/${INSTALL_INCLUDE_DIR})

install(FILES ${harness_headers}
  DESTINATION ${HARNESS_INSTALL_INCLUDE_PREFIX}/${HARNESS_NAME})
if(NOT WIN32)
  install(TARGETS harness-archive harness-library
        LIBRARY DESTINATION ${HARNESS_INSTALL_LIBRARY_DIR}
        ARCHIVE DESTINATION ${HARNESS_INSTALL_LIBRARY_DIR})
else()
  install(TARGETS harness-archive
        ARCHIVE DESTINATION ${HARNESS_INSTALL_LIBRARY_DIR})
  install(TARGETS harness-library
        RUNTIME DESTINATION ${HARNESS_INSTALL_BIN_DIR}
        LIBRARY DESTINATION ${HARNESS_INSTALL_LIBRARY_DIR})          
endif()

if(ENABLE_TESTS)
  add_subdirectory(tests)
endif()
