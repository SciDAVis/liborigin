# CMakeLists.txt for liborigin
cmake_minimum_required(VERSION 3.16.3)

project( liborigin
  VERSION 3.0.0
  LANGUAGES CXX
  )

set( CMAKE_CXX_STANDARD 17 )
set( CMAKE_CXX_STANDARD_REQUIRED TRUE )
set( CMAKE_CXX_EXTENSIONS OFF )

if( MSVC )
  # /wd4456 /wd4458 Silent "declaration of %1 hides %2 ..."
  set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHsc /W4 /permissive- /wd4456 /wd4458" )
  add_compile_definitions( _CRT_SECURE_NO_WARNINGS
    _SILENCE_CXX17_OLD_ALLOCATOR_MEMBERS_DEPRECATION_WARNING )
  if( CMAKE_CXX_COMPILER_ID STREQUAL "Clang" )
    set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-nonportable-include-path" )
  else()
    add_compile_options( "/MP" )
  endif()
else()
  set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fexceptions -Wall -Wextra -pedantic" )
  set( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -pedantic" )
endif()

# compile-time configuration variables to be linked in
configure_file(config.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h)
include_directories(${CMAKE_CURRENT_BINARY_DIR})

# source files
set (sources
	OriginFile.cpp
	OriginParser.cpp
	OriginAnyParser.cpp
	)

# header files for development
set (devel-headers
	OriginObj.h
	OriginFile.h
	OriginParser.h
	tree.hh
	)

# Generate a pkg-config file matching this config
if( "${CMAKE_SOURCE_DIR}" STREQUAL "${PROJECT_SOURCE_DIR}" ) 
  configure_file( "liborigin.pc.in" "$liborigin.pc" @ONLY )
endif()

# allow for installation of individual targets
set(CMAKE_SKIP_INSTALL_ALL_DEPENDENCY true)

# object library
add_library (origin OBJECT ${sources})
set_target_properties(origin PROPERTIES POSITION_INDEPENDENT_CODE ON)

# inform on log setting
if (DEFINED GENERATE_CODE_FOR_LOG)
    message("File parsing will be logged.")
    target_compile_definitions(origin PRIVATE GENERATE_CODE_FOR_LOG=${GENERATE_CODE_FOR_LOG})
else()
    message("File parsing will NOT be logged. Define GENERATE_CODE_FOR_LOG to activate logging.")
endif ()

# Shared library
add_library( origin_shared SHARED $<TARGET_OBJECTS:origin>)

# Static library
add_library (origin_static STATIC $<TARGET_OBJECTS:origin>)

set_target_properties(origin_shared origin_static PROPERTIES
    OUTPUT_NAME "origin"
	VERSION ${PROJECT_VERSION}
	SOVERSION ${PROJECT_VERSION_MAJOR}
    )

# command line util
add_executable(opj2dat opj2dat.cpp)
target_link_libraries (opj2dat origin_static)

include(GNUInstallDirs)

if( "${CMAKE_SOURCE_DIR}" STREQUAL "${PROJECT_SOURCE_DIR}" )
  # install libraries
  install( TARGETS origin_shared origin_static
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    )
  # install headers
  install(FILES ${devel-headers} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/liborigin)
  # install pkg-config file
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/liborigin.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
  #
  install(TARGETS opj2dat DESTINATION ${CMAKE_INSTALL_BINDIR} OPTIONAL)
  # documentation
  install(FILES COPYING FORMAT README
    DESTINATION ${CMAKE_INSTALL_DOCDIR}
    OPTIONAL
    )
  # Doxygen
  configure_file(doc/Doxyfile.in ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile)
  find_package(Doxygen)
  if(DOXYGEN_FOUND)
	add_custom_target(doc ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
		WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
	set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES ${CMAKE_CURRENT_BINARY_DIR}/html)
	install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/html
      DESTINATION ${CMAKE_INSTALL_DOCDIR}
      OPTIONAL
      )
  endif()
endif()
