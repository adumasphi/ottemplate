cmake_minimum_required ( VERSION 2.8 )

option ( BUILD_PYTHON                 "Build the python module for the library"                               ON )
option (USE_SPHINX                    "Use sphinx for documentation"                                          ON)
option ( LINK_PYTHON_LIBRARY          "Link python modules against python library"                            ON )
option ( BUILD_SHARED_LIBS            "Build shared libraries"                                                ON )

# By default, build in Release mode. Must appear before project() command
if ( NOT DEFINED CMAKE_BUILD_TYPE )
   set( CMAKE_BUILD_TYPE Release CACHE STRING "Build type")
endif ()

project ( OTTemplate CXX )

string ( TOLOWER ${CMAKE_PROJECT_NAME} PACKAGE_NAME )

# set module dir to find custom scripts
list ( APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake )

find_package (OpenTURNS NO_MODULE REQUIRED)
message (STATUS "Found OpenTURNS: ${OPENTURNS_ROOT_DIR} (found version \"${OPENTURNS_VERSION_STRING}\")")
include ( ${OPENTURNS_USE_FILE} )
set ( OTTEMPLATE_DEFINITIONS ${OPENTURNS_DEFINITIONS} )
if (NOT BUILD_SHARED_LIBS)
  list ( APPEND OTTEMPLATE_DEFINITIONS "-DOTTEMPLATE_STATIC" )
endif ()
add_definitions ( ${OTTEMPLATE_DEFINITIONS} )

set ( OTTEMPLATE_LIBRARY_PATH lib${LIB_SUFFIX} )
set ( OTTEMPLATE_INCLUDE_PATH include)
set ( OTTEMPLATE_SWIG_INCLUDE_DIRS ${OTTEMPLATE_INCLUDE_PATH}/${PACKAGE_NAME}/swig )
set ( OTTEMPLATE_DATA_PATH share )
set ( OTTEMPLATE_EXAMPLE_PATH ${OTTEMPLATE_DATA_PATH}/${PACKAGE_NAME}/examples )
set ( OTTEMPLATE_CONFIG_CMAKE_PATH lib${LIB_SUFFIX}/cmake/ottemplate )
set (OTTEMPLATE_DOC_PATH share/${PACKAGE_NAME}/doc)

if (BUILD_PYTHON)
  # workaround for cmake bug #0013449
  if (CMAKE_VERSION VERSION_LESS 3 AND DEFINED CMAKE_FIND_ROOT_PATH)
    find_program (SWIG_EXECUTABLE NAMES swig2.0 swig)
    if (SWIG_EXECUTABLE)
      set (SWIG_USE_FILE ${CMAKE_ROOT}/Modules/UseSWIG.cmake)
      set (SWIG_FOUND TRUE)
    endif ()
  else ()
    find_package (SWIG)
  endif ()
  if (SWIG_FOUND)
    include (${SWIG_USE_FILE})
  endif ()

  find_package ( PythonInterp )
  find_package ( PythonLibs )
  include (FindPythonModule)

  if (USE_SPHINX)
    find_program (SPHINX_EXECUTABLE NAMES sphinx-build DOC "Sphinx Documentation Builder (sphinx-doc.org)")
    if (SPHINX_EXECUTABLE)
      find_python_module (sphinx 1.1)
      find_python_module (numpydoc)
    endif ()
  endif ()

  if ( PYTHONINTERP_FOUND )
    if (DEFINED PYTHON_SITE_PACKAGES)
      set (OTTEMPLATE_PYTHON_MODULE_PATH ${PYTHON_SITE_PACKAGES})

    else ()
      execute_process ( COMMAND ${PYTHON_EXECUTABLE} -c "from distutils import sysconfig; print(sysconfig.get_python_lib(plat_specific=True, prefix='${CMAKE_INSTALL_PREFIX}'))"
                        OUTPUT_VARIABLE _ABS_PYTHON_MODULE_PATH
                        RESULT_VARIABLE _PYTHON_pythonlib_result
                        OUTPUT_STRIP_TRAILING_WHITESPACE )

      if (_PYTHON_pythonlib_result)
        message (SEND_ERROR "Could not run ${PYTHON_EXECUTABLE}")
      endif ()

      get_filename_component (_ABS_PYTHON_MODULE_PATH ${_ABS_PYTHON_MODULE_PATH} ABSOLUTE)
      file (RELATIVE_PATH OTTEMPLATE_PYTHON_MODULE_PATH ${CMAKE_INSTALL_PREFIX} ${_ABS_PYTHON_MODULE_PATH})
    endif ()

    if (IS_ABSOLUTE ${OTTEMPLATE_PYTHON_MODULE_PATH})
      set (PYTHON_MODULE_PATH ${OTTEMPLATE_PYTHON_MODULE_PATH})
    else ()
      set (PYTHON_MODULE_PATH ${CMAKE_INSTALL_PREFIX}/${OTTEMPLATE_PYTHON_MODULE_PATH})
    endif ()

    if (CMAKE_VERSION VERSION_LESS "2.8.6")
      execute_process (COMMAND ${PYTHON_EXECUTABLE} -c "import sys; print(sys.version_info[0])"
                       OUTPUT_VARIABLE PYTHON_VERSION_MAJOR
                       OUTPUT_STRIP_TRAILING_WHITESPACE)
    endif ()
    set (OTTEMPLATE_PYTHON${PYTHON_VERSION_MAJOR}_MODULE_PATH ${OTTEMPLATE_PYTHON_MODULE_PATH})
  endif ( PYTHONINTERP_FOUND )
endif ()


# WARNING: This is where variables without OTTEMPLATE_ prefix are defined !
foreach (_var PYTHON_MODULE_PATH LIBRARY_PATH INCLUDE_PATH CONFIG_CMAKE_PATH)
  file (TO_NATIVE_PATH "${OTTEMPLATE_${_var}}" NATIVE_OTTEMPLATE_${_var})
  if (NOT IS_ABSOLUTE ${NATIVE_OTTEMPLATE_${_var}})
    file (TO_NATIVE_PATH ${CMAKE_INSTALL_PREFIX}/${OTTEMPLATE_${_var}} NATIVE_OTTEMPLATE_${_var})
  endif ()
  string (REPLACE "\\" "\\\\" ${_var} ${NATIVE_OTTEMPLATE_${_var}})
endforeach ()

# RPATH settings
set (CMAKE_INSTALL_RPATH ${LIBRARY_PATH})
set (CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
if(POLICY CMP0042)
  # Set MACOSX_RPATH to ON
  cmake_policy(SET CMP0042 NEW)
endif()

# Some useful macros to ease CMakeLists.txt file writing
set ( SOURCEFILES "" CACHE INTERNAL "List of source files to compile" )
macro ( ot_add_source_file FILENAME )
  set ( sf ${SOURCEFILES} ${CMAKE_CURRENT_SOURCE_DIR}/${FILENAME} )
  set ( SOURCEFILES ${sf} CACHE INTERNAL "List of source files to compile" )
endmacro ( ot_add_source_file )

set ( HEADERFILES "" CACHE INTERNAL "List of header files to install" )
macro ( ot_install_header_file FILENAME )
  set ( hf ${HEADERFILES} ${CMAKE_CURRENT_SOURCE_DIR}/${PACKAGE_NAME}/${FILENAME} )
  set ( HEADERFILES ${hf} CACHE INTERNAL "List of header files to install" )
endmacro ( ot_install_header_file )

set ( SWIGFILES "" CACHE INTERNAL "List of SWIG files to install" )
macro ( ot_install_swig_file FILENAME )
  set ( hf ${SWIGFILES} ${CMAKE_CURRENT_SOURCE_DIR}/${FILENAME} )
  set ( SWIGFILES ${hf} CACHE INTERNAL "List of SWIG files to install" )
endmacro ( ot_install_swig_file )

set ( INTERNAL_INCLUDE_DIRS "" CACHE INTERNAL "List of directories with header files needed for build" )
macro ( ot_add_current_dir_to_include_dirs )
  set ( inc_dirs ${INTERNAL_INCLUDE_DIRS} ${CMAKE_CURRENT_SOURCE_DIR} )
  set ( INTERNAL_INCLUDE_DIRS ${inc_dirs} CACHE INTERNAL "List of directories with header files needed for build" )
endmacro ( ot_add_current_dir_to_include_dirs )


set ( CPACK_PACKAGE_NAME          ${PACKAGE_NAME} )
set ( CPACK_PACKAGE_VERSION_MAJOR 0              )
set ( CPACK_PACKAGE_VERSION_MINOR 0              )
set ( CPACK_PACKAGE_VERSION_PATCH                )
set ( CPACK_SOURCE_GENERATOR      "TGZ;TBZ2"     )
set ( CPACK_BINARY_STGZ           "OFF"          )
set ( CPACK_BINARY_TBZ2           "ON"           )
set ( CPACK_BINARY_TGZ            "ON"           )
set ( CPACK_BINARY_TZ             "OFF"          )
set ( CPACK_SOURCE_IGNORE_FILES "/.git;/build;.*~;${CPACK_SOURCE_IGNORE_FILES}" )
set ( PACKAGE_NAME           ${CPACK_PACKAGE_NAME} )
set ( PACKAGE_VERSION        ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR} )
if ( CPACK_PACKAGE_VERSION_PATCH )
  set ( PACKAGE_VERSION       ${PACKAGE_VERSION}.${CPACK_PACKAGE_VERSION_PATCH} )
endif ()
set ( CPACK_SOURCE_PACKAGE_FILE_NAME ${PACKAGE_NAME}-${PACKAGE_VERSION} )


# The tests can't be run if this function is absent
enable_testing ()
add_custom_target (tests COMMENT "Build tests")
add_custom_target (check COMMENT "Run pre-installation tests")
add_custom_target (installcheck COMMENT "Run post-installation tests")

add_subdirectory (lib)
add_dependencies (check cppcheck)

if (PYTHONINTERP_FOUND AND PYTHONLIBS_FOUND AND SWIG_FOUND AND BUILD_SHARED_LIBS)
  add_subdirectory (python)
  add_dependencies (installcheck pyinstallcheck)
endif ()


# uninstall target
configure_file (
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
  IMMEDIATE @ONLY
)

add_custom_target ( uninstall
  COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
)

set ( OTTEMPLATE_LIBRARY                 ottemplate )
set ( OTTEMPLATE_LIBRARIES ${OPENTURNS_LIBRARIES} )
list ( APPEND OTTEMPLATE_LIBRARIES ${OTTEMPLATE_LIBRARY} )
set ( OTTEMPLATE_LIBRARY_DIRS ${OPENTURNS_LIBRARY_DIRS} )
list ( APPEND OTTEMPLATE_LIBRARY_DIRS ${LIBRARY_PATH} )
set ( OTTEMPLATE_INCLUDE_DIR ${INCLUDE_PATH} )
set ( OTTEMPLATE_INCLUDE_DIRS ${OTTEMPLATE_INCLUDE_DIR} )
set ( OTTEMPLATE_ROOT_DIR       ${CMAKE_INSTALL_PREFIX} )
list ( APPEND OTTEMPLATE_INCLUDE_DIRS ${OPENTURNS_INCLUDE_DIRS} )
set ( OTTEMPLATE_VERSION_STRING ${PACKAGE_VERSION} )
set ( OTTEMPLATE_VERSION_MAJOR ${CPACK_PACKAGE_VERSION_MAJOR} )
set ( OTTEMPLATE_VERSION_MINOR ${CPACK_PACKAGE_VERSION_MINOR} )
set ( OTTEMPLATE_VERSION_PATCH ${CPACK_PACKAGE_VERSION_PATCH} )
set ( OTTEMPLATE_USE_FILE      ${CONFIG_CMAKE_PATH}/UseOTTemplate.cmake )

if (CMAKE_VERSION VERSION_LESS 2.8.8)
  # CMakePackageConfigHelpers is not available
  set (PACKAGE_INIT "
# Generated by a CMake version without CMakePackageConfigHelpers.
macro(set_and_check _var _value)
  set(\${_var} \"\${_value}\")
endmacro(set_and_check _var _value)
"     )
  foreach (_var
      OTTEMPLATE_USE_FILE
      OTTEMPLATE_ROOT_DIR
      OTTEMPLATE_INCLUDE_DIR
      OTTEMPLATE_INCLUDE_DIRS
      OTTEMPLATE_LIBRARY_DIRS )
    set(PACKAGE_${_var} ${${_var}})
  endforeach ()
  
configure_file ( ${CMAKE_SOURCE_DIR}/cmake/OTTemplateConfig.cmake.in
                 ${CMAKE_BINARY_DIR}/OTTemplateConfig.cmake
                 @ONLY ESCAPE_QUOTES
               )

install ( FILES cmake/UseOTTemplate.cmake 
                ${CMAKE_BINARY_DIR}/OTTemplateConfig.cmake
          DESTINATION ${OTTEMPLATE_CONFIG_CMAKE_PATH}
        )

else ()

  include(CMakePackageConfigHelpers)
  configure_package_config_file (
    cmake/OTTemplateConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/OTTemplateConfig.cmake
    INSTALL_DESTINATION ${OTTEMPLATE_CONFIG_CMAKE_PATH}
    PATH_VARS
      OTTEMPLATE_USE_FILE
      OTTEMPLATE_ROOT_DIR
      OTTEMPLATE_INCLUDE_DIR
      OTTEMPLATE_INCLUDE_DIRS
      OTTEMPLATE_LIBRARY_DIRS
    NO_CHECK_REQUIRED_COMPONENTS_MACRO
                                )                   
  write_basic_package_version_file( OTTemplateConfigVersion.cmake VERSION ${OTTEMPLATE_VERSION_STRING} COMPATIBILITY AnyNewerVersion )               

  install ( FILES cmake/UseOTTemplate.cmake
                  ${CMAKE_CURRENT_BINARY_DIR}/OTTemplateConfig.cmake
                  ${CMAKE_CURRENT_BINARY_DIR}/OTTemplateConfigVersion.cmake
            DESTINATION ${OTTEMPLATE_CONFIG_CMAKE_PATH}
          )
endif ()

include (CPack)
