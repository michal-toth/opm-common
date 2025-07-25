cmake_minimum_required (VERSION 3.10)
project(opm-common C CXX)

option(SIBLING_SEARCH "Search for other modules in sibling directories?" ON)
list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/Modules)
set(OPM_MACROS_ROOT ${PROJECT_SOURCE_DIR})

option(ENABLE_ECL_INPUT "Enable eclipse input support?" ON)
option(ENABLE_ECL_OUTPUT "Enable eclipse output support?" ON)
option(ENABLE_MOCKSIM "Build the mock simulator for io testing" ON)
option(OPM_ENABLE_PYTHON "Enable python bindings?" OFF)
option(OPM_INSTALL_PYTHON "Install python bindings?" ON)
option(OPM_ENABLE_EMBEDDED_PYTHON "Enable embedded python?" OFF)
option(OPM_ENABLE_DUNE "Enable code requiring dune-common?" ON)

# Output implies input
if(ENABLE_ECL_OUTPUT)
  set(ENABLE_ECL_INPUT ON)
endif()

# And likewise, no input means no output
if(NOT ENABLE_ECL_INPUT)
  set(ENABLE_ECL_OUTPUT OFF)
endif()


# not the same location as most of the other projects; this hook overrides
macro (dir_hook)
endmacro (dir_hook)

# We need to define this variable in the installed cmake config file.
set(OPM_PROJECT_EXTRA_CODE_INSTALLED  "#ENABLE_ECL_INPUT is needed by opm-common-prereq.cmake
                                       set(ENABLE_ECL_INPUT ${ENABLE_ECL_INPUT})
                                       set(OPM_MACROS_ROOT ${CMAKE_INSTALL_PREFIX}/share/opm)
                                       list(APPEND CMAKE_MODULE_PATH \${OPM_MACROS_ROOT}/cmake/Modules)
                                       include(OpmPackage) #Make macros available after find_package(opm-common)")

set(OPM_PROJECT_EXTRA_CODE_INTREE "#ENABLE_ECL_INPUT is needed by opm-common-prereq.cmake
                                   set(ENABLE_ECL_INPUT ${ENABLE_ECL_INPUT})
                                   set(OPM_MACROS_ROOT ${OPM_MACROS_ROOT})
                                   list(APPEND CMAKE_MODULE_PATH \${OPM_MACROS_ROOT}/cmake/Modules)
                                   include(OpmPackage) #Make macros available after find_package(opm-common)")
if(ENABLE_ECL_OUTPUT)
  set(OPM_PROJECT_EXTRA_CODE_INSTALLED "${OPM_PROJECT_EXTRA_CODE_INSTALLED}
                                        set(COMPARE_ECL_COMMAND ${CMAKE_INSTALL_PREFIX}/bin${${name}_VER_DIR}/compareECL)
                                        set(OPM_PACK_COMMAND ${CMAKE_INSTALL_PREFIX}/bin${${name}_VER_DIR}/opmpack)
                                        set(RST_DECK_COMMAND ${CMAKE_INSTALL_PREFIX}/bin${${name}_VER_DIR}/rst_deck)")

  set(OPM_PROJECT_EXTRA_CODE_INTREE "${OPM_PROJECT_EXTRA_CODE_INTREE}
                                     set(COMPARE_ECL_COMMAND ${PROJECT_BINARY_DIR}/bin/compareECL)
                                     set(OPM_PACK_COMMAND ${PROJECT_BINARY_DIR}/bin/opmpack)
                                     set(RST_DECK_COMMAND ${PROJECT_BINARY_DIR}/bin/rst_deck)")
endif()

# project information is in dune.module. Read this file and set variables.
# we cannot generate dune.module since it is read by dunecontrol before
# the build starts, so it makes sense to keep the data there then.
include (OpmInit)
OpmSetPolicies()

# Look for the opm-tests repository; if found the variable
# HAVE_OPM_TESTS will be set to true.
include(Findopm-tests)

# list of prerequisites for this particular project; this is in a
# separate file (in cmake/Modules sub-directory) because it is shared
# with the find module
include (${project}-prereqs)

# source_hook runs before config_hook and the former needs fmt, hence this
# needs to be here.
if(fmt_FOUND)
  # OpmSatellites will not add the library, do it here.
  list(APPEND opm-common_LIBRARIES fmt::fmt)
else()
  include(DownloadFmt)
endif()

if(OPM_ENABLE_EMBEDDED_PYTHON AND NOT OPM_ENABLE_PYTHON)
  # This needs to be here to run before source_hook
  message(WARNING "Inconsistent settings: OPM_ENABLE_PYTHON=OFF and "
    "OPM_ENABLE_EMBEDDED_PYTHON=ON. OPM_ENABLE_EMBEDDED_PYTHON=ON now implies OPM_ENABLE_PYTHON=ON.")
  set(OPM_ENABLE_PYTHON ON CACHE BOOL "Enable python bindings?" FORCE)
endif()
# read the list of components from this file (in the project directory);
# it should set various lists with the names of the files to include
include (CMakeLists_files.cmake)

macro (config_hook)
  if(ENABLE_ECL_INPUT)
    # For this project
    include_directories(${EXTRA_INCLUDES} ${PROJECT_BINARY_DIR}/include)
    # For downstreams
    list(APPEND EXTRA_INCLUDES ${PROJECT_BINARY_DIR}/include)
    set(OPM_PROJECT_EXTRA_CODE_INTREE "${OPM_PROJECT_EXTRA_CODE_INTREE}
                                       list(APPEND opm-common_INCLUDE_DIRS ${EXTRA_INCLUDES})")
    if(ENABLE_ECL_INPUT)
      set(OPM_PROJECT_EXTRA_CODE_INTREE "${OPM_PROJECT_EXTRA_CODE_INTREE}
                                         set(HAVE_ECL_INPUT 1)")
      set(OPM_PROJECT_EXTRA_CODE_INSTALLED "${OPM_PROJECT_EXTRA_CODE_INSTALLED}
                                            set(HAVE_ECL_INPUT 1)")
      set(HAVE_ECL_INPUT 1)
    endif()
    if(ENABLE_ECL_OUTPUT)
      set(OPM_PROJECT_EXTRA_CODE_INTREE "${OPM_PROJECT_EXTRA_CODE_INTREE}
                                         set(HAVE_ECL_OUTPUT 1)")
      set(OPM_PROJECT_EXTRA_CODE_INSTALLED "${OPM_PROJECT_EXTRA_CODE_INSTALLED}
                                            set(HAVE_ECL_OUTPUT 1)")
    endif()
    # We need to use the correct search mode. Otherwise not finding one
    # boost component beloq will mark the previously found ones as not
    # found again. (Code stolen from UseDynamicBoost.cmake
    if(Boost_DIR)
      set(_Boost_CONFIG_MODE CONFIG)
    endif()
    find_package(Boost COMPONENTS filesystem regex system unit_test_framework ${_Boost_CONFIG_MODE})

    if (HAVE_DYNAMIC_BOOST_TEST)
      set_target_properties(Boost::unit_test_framework PROPERTIES INTERFACE_COMPILE_DEFINITIONS BOOST_TEST_DYN_LINK=1)
    endif()
  endif()

  include(CheckIncludeFile)
  check_include_file(fnmatch.h FNMATCH_H_FOUND)
  if (FNMATCH_H_FOUND)
    set(HAVE_FNMATCH_H 1)
  endif()

  if(OPM_ENABLE_DUNE)
    find_package(dune-common REQUIRED)
    opm_need_version_of ("dune-common")
  endif()
endmacro (config_hook)

macro (prereqs_hook)
endmacro (prereqs_hook)

macro (sources_hook)
  if(NOT cjson_FOUND)
    include(DownloadCjson)
    include_directories(${cjson_SOURCE_DIR})
    list(APPEND opm-common_SOURCES ${cjson_SOURCE_DIR}/cJSON.c)
  endif()

  if(ENABLE_ECL_INPUT)
    # Keyword generation
    include(GenerateKeywords.cmake)

    # Append generated sources
    list(INSERT opm-common_SOURCES 0 ${PROJECT_BINARY_DIR}/ParserInit.cpp)
    foreach (name A B C D E F G H I J K L M N O P Q R S T U V W X Y Z)
        list(INSERT opm-common_SOURCES 0 ${PROJECT_BINARY_DIR}/ParserKeywords/${name}.cpp)
        list(INSERT opm-common_SOURCES 0 ${PROJECT_BINARY_DIR}/ParserKeywords/ParserInit${name}.cpp)
        list(INSERT opm-common_SOURCES 0 ${PROJECT_BINARY_DIR}/ParserKeywords/Builtin${name}.cpp)
        list(INSERT opm-common_HEADERS 0 ${PROJECT_BINARY_DIR}/include/opm/input/eclipse/Parser/ParserKeywords/${name}.hpp)
    endforeach()
    if (OPM_ENABLE_EMBEDDED_PYTHON)
      list(INSERT opm-common_SOURCES 0 ${PROJECT_BINARY_DIR}/python/cxx/builtin_pybind11.cpp)
    endif()
  endif()
  set_source_files_properties(src/opm/input/eclipse/Python/Python.cpp
                              PROPERTIES COMPILE_FLAGS -Wno-shadow)
  if (OPM_ENABLE_PYTHON)
    # Set the path to the input docstrings.json file and the output .hpp file
    set(PYTHON_DOCSTRINGS_FILE "${PROJECT_SOURCE_DIR}/python/docstrings_common.json")
    set(PYTHON_DOCSTRINGS_GENERATED_HPP "${PROJECT_BINARY_DIR}/python/cxx/OpmCommonPythonDoc.hpp")
    # Command to run the Python script
    add_custom_command(
        OUTPUT ${PYTHON_DOCSTRINGS_GENERATED_HPP}
        COMMAND ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/python/generate_docstring_hpp.py
                ${PYTHON_DOCSTRINGS_FILE} ${PYTHON_DOCSTRINGS_GENERATED_HPP} OPMCOMMONPYTHONDOC_HPP "Opm::Common::DocStrings"
        DEPENDS ${PYTHON_DOCSTRINGS_FILE}
        COMMENT "Generating OpmCommonPythonDoc.hpp from JSON file"
    )
    list(INSERT opm-common_SOURCES 0 ${PYTHON_DOCSTRINGS_GENERATED_HPP})
  endif()

  if(QuadMath_FOUND)
    get_target_property(qm_defs QuadMath::QuadMath INTERFACE_COMPILE_DEFINITIONS)
    list(APPEND qm_defs HAVE_QUAD=1)
    get_target_property(qm_options QuadMath::QuadMath INTERFACE_COMPILE_OPTIONS)
    set_source_files_properties(opm/material/components/CO2.cpp
                                opm/material/densead/Evaluation.cpp
                                PROPERTIES COMPILE_DEFINITIONS "${qm_defs}"
                                COMPILE_OPTIONS "${qm_options}")
  endif()
endmacro (sources_hook)

macro (fortran_hook)
endmacro (fortran_hook)

macro (files_hook)
endmacro (files_hook)

macro (tests_hook)
  if(ENABLE_ECL_INPUT)
    include(ExtraTests.cmake)
  endif()
  if (dune-common_FOUND)
    list(APPEND tests_SOURCES ${DUNE_TEST_SOURCE_FILES})
  endif()
endmacro (tests_hook)

macro (install_hook)
  install(DIRECTORY ${PROJECT_BINARY_DIR}/include/
          DESTINATION include
          PATTERN *.hpp)
endmacro (install_hook)

# Used to append entries from one list to another.
# The output list is suitable for use in setup.py subtitution
macro(append_quoted OUT IN)
  foreach(ENTRY ${${IN}})
    list(APPEND ${OUT} "'${ARGN}${ENTRY}'")
  endforeach()
endmacro()

# If opm-common is configured to embed the python interpreter we must make sure
# that all downstream modules link libpython transitively. Due to the required
# integration with Python+cmake machinery provided by pybind11 this is done by
# manually adding to the opm-common_LIBRARIES variable here, and not in the
# OpmLibMain function. Here only the library dependency is implemented, the
# bulk of the python configuration is further down in the file.
if (OPM_ENABLE_PYTHON)
  # We need to be compatible with older CMake versions
  # that do not offer FindPython3
  # e.g. Ubuntu LTS 18.04 uses cmake 3.10
  if(${CMAKE_VERSION} VERSION_LESS "3.12.0")
    find_package(PythonInterp REQUIRED)
    if (OPM_ENABLE_EMBEDDED_PYTHON)
      find_package(PythonLibs REQUIRED)
      list(APPEND opm-common_LIBRARIES ${PYTHON_LIBRARIES})
    endif()
    if(PYTHON_VERSION_MAJOR LESS 3)
      message(SEND_ERROR "OPM requires version 3 of Python but only version ${PYTHON_VERSION_STRING} was found")
    endif()
    set(Python3_EXECUTABLE ${PYTHON_EXECUTABLE})
    set(Python3_LIBRARIES ${PYTHON_LIBRARIES})
    set(Python3_VERSION "${PYTHON_VERSION_STRING}")
    set(Python3_VERSION_MINOR ${PYTHON_VERSION_MINOR})
  else()
    # Be backwards compatible.
    if(PYTHON_EXECUTABLE AND NOT Python3_EXECUTABLE)
      set(Python3_EXECUTABLE ${PYTHON_EXECUTABLE})
    endif()
    # We always need to search for Development as we use
    # pybind11_add_module even if don't embed Python
    if (NOT OPM_ENABLE_EMBEDDED_PYTHON)
      if(${CMAKE_VERSION} VERSION_LESS "3.18.0")
        find_package(Python3 REQUIRED COMPONENTS Interpreter Development)
      else()
        find_package(Python3 REQUIRED COMPONENTS Interpreter Development.Module)
      endif()
    else()
      if(${CMAKE_VERSION} VERSION_LESS "3.18.0")
        find_package(Python3 REQUIRED COMPONENTS Interpreter Development)
      else()
        find_package(Python3 REQUIRED COMPONENTS Interpreter Development.Embed Development.Module)
      endif()
      get_target_property(_lib_path Python3::Python IMPORTED_LOCATION)
      set(PYTHON_LIBRARY ${_lib_path})
      set(PYTHON_LIBRARIES {PYTHON_LIBRARY})
      list(APPEND opm-common_LIBRARIES ${PYTHON_LIBRARY})
      set(PYTHON_INCLUDE_DIRS ${Python3_INCLUDE_DIRS})
    endif()
    if(Python3_VERSION_MINOR LESS 3)
      # Python native namespace packages requires python >= 3.3
      message(SEND_ERROR "OPM requires python >= 3.3 but only version ${Python3_VERSION} was found")
    endif()

    # Compatibility settings for PythonInterp and PythonLibs
    # used e.g. in FindCwrap, pybind11
    set(PYTHON_EXECUTABLE ${Python3_EXECUTABLE})
    # Directory to install common (for opm modules) python scripts
    include (GNUInstallDirs)
    set(OPM_PYTHON_COMMON_DIR "${CMAKE_INSTALL_DATAROOTDIR}/opm/python")
    set(OPM_PROJECT_EXTRA_CODE_INTREE "${OPM_PROJECT_EXTRA_CODE_INTREE}
                                       set(opm-common_PYTHON_COMMON_DIR ${PROJECT_SOURCE_DIR}/python)")
    set(OPM_PROJECT_EXTRA_CODE_INSTALLED "${OPM_PROJECT_EXTRA_CODE_INSTALLED}
                                         set(opm-common_PYTHON_COMMON_DIR ${CMAKE_INSTALL_PREFIX}/${OPM_PYTHON_COMMON_DIR})")
  endif()
  # We always need the PYTHON_INCLUDE_DIR. Unfortunately
  # When we build pypi packages CMake will fail to determine
  # these via the usual find_package(PythonLibs or
  # find_package(Python3 REQUIRED COMPONENTS Interpreter Development)
  # Hence we overwrite them here.
  if(NOT PYTHON_INCLUDE_DIRS)
    execute_process(COMMAND "${PYTHON_EXECUTABLE}" "-c" "from distutils import sysconfig; print(sysconfig.get_python_inc(plat_specific=True), end=\"\");"
      RESULT_VARIABLE _PYTHON_DIR_SUCCESS
      OUTPUT_VARIABLE PYTHON_INCLUDE_DIR
      ERROR_VARIABLE _PYTHON_ERROR_VALUE)
    if(NOT _PYTHON_DIR_SUCCESS MATCHES 0)
      message(FATAL_ERROR "Could not determine Python include directory. Error: ${_PYTHON_ERROR_VALUE}.")
    endif()
    set(PYTHON_INCLUDE_DIRS ${PYTHON_INCLUDE_DIR})
  endif()
  find_package(pybind11 2.2 CONFIG)
  if (NOT pybind11_FOUND)
    include(DownloadPyBind11)
  endif()
endif()


# all setup common to the OPM library modules is done here
include (OpmLibMain)

if (ENABLE_MOCKSIM AND ENABLE_ECL_INPUT)
  add_library(mocksim
              msim/src/msim.cpp)
  target_link_libraries(mocksim opmcommon)
  target_include_directories(mocksim PUBLIC msim/include)
  add_executable(msim examples/msim.cpp)
  target_link_libraries(msim mocksim)

  if (Boost_UNIT_TEST_FRAMEWORK_FOUND)
    set(_libs mocksim opmcommon
      ${Boost_UNIT_TEST_FRAMEWORK_LIBRARY})

    foreach( test test_msim test_msim_ACTIONX test_msim_EXIT)
      opm_add_test(${test} SOURCES tests/msim/${test}.cpp
        LIBRARIES ${_libs}
        WORKING_DIRECTORY ${PROJECT_BINARY_DIR}/tests
        CONDITION ${HAVE_ECL_INPUT})
    endforeach()
    set_tests_properties(msim_ACTIONX PROPERTIES
                        ENVIRONMENT "PYTHONPATH=${PROJECT_BINARY_DIR}/python:$ENV{PYTHONPATH}")
  endif()
endif()

# Build the compare utilities
if(ENABLE_ECL_INPUT)
  add_executable(compareECL
    test_util/EclFilesComparator.cpp
    test_util/EclRegressionTest.cpp
    test_util/compareECL.cpp
    )

  add_executable(convertECL
    test_util/convertECL.cpp
    )

  add_executable(summary
    test_util/summary.cpp
    )

  add_executable(arraylist
    test_util/arraylist.cpp
    )

  add_executable(rewriteEclFile
    test_util/rewriteEclFile.cpp
    )

  foreach(target compareECL convertECL summary rewriteEclFile arraylist)
    target_link_libraries(${target} opmcommon)
    install(TARGETS ${target} DESTINATION bin)
  endforeach()

  # Add the tests
  set(_libs opmcommon
            ${Boost_UNIT_TEST_FRAMEWORK_LIBRARY})

  opm_add_test(test_EclFilesComparator
    CONDITION
      ENABLE_ECL_INPUT AND Boost_UNIT_TEST_FRAMEWORK_FOUND
    SOURCES
      tests/test_EclFilesComparator.cpp
      test_util/EclFilesComparator.cpp
    LIBRARIES
      ${_libs}
    WORKING_DIRECTORY
      ${PROJECT_BINARY_DIR}/tests
    )

  opm_add_test(test_EclRegressionTest
    CONDITION
      ENABLE_ECL_INPUT AND Boost_UNIT_TEST_FRAMEWORK_FOUND
    SOURCES
      tests/test_EclRegressionTest.cpp
      test_util/EclFilesComparator.cpp
      test_util/EclRegressionTest.cpp
    LIBRARIES
      ${_libs}
    WORKING_DIRECTORY
      ${PROJECT_BINARY_DIR}/tests
    )

endif()

if(dune-common_FOUND)
  # Explicitly link tests needing dune-common.
  # To avoid pulling dune-common into the opm-common interface
  target_include_directories(dunecommon INTERFACE ${dune-common_INCLUDE_DIRS})
  target_include_directories(opmcommon PRIVATE ${dune-common_INCLUDE_DIRS})
  string(REPLACE " " ";" dflags "${dune-common_CXX_FLAGS}")
  target_compile_options(dunecommon INTERFACE ${dflags})
  target_compile_definitions(dunecommon INTERFACE DUNE_COMMON_VERSION_MAJOR=${DUNE_COMMON_VERSION_MAJOR})
  target_compile_definitions(dunecommon INTERFACE DUNE_COMMON_VERSION_MINOR=${DUNE_COMMON_VERSION_MINOR})
  target_compile_definitions(dunecommon INTERFACE DUNE_COMMON_VERSION_REVISION=${DUNE_COMMON_VERSION_REVISION})
  if(Boost_UNIT_TEST_FRAMEWORK_FOUND)
    foreach(src ${DUNE_TEST_SOURCE_FILES})
      get_filename_component(tgt ${src} NAME_WE)
      target_link_libraries(${tgt} dunecommon)
    endforeach()
    target_link_libraries(test_SymmTensor dunecommon)
  endif()
  if(BUILD_EXAMPLES)
    target_link_libraries(co2brinepvt dunecommon)
  endif()
endif()

if(Boost_UNIT_TEST_FRAMEWORK_FOUND)
  foreach(test ACTIONX EmbeddedPython PYACTION)
    set_tests_properties(${test} PROPERTIES
                         ENVIRONMENT "PYTHONPATH=${PROJECT_BINARY_DIR}/python:$ENV{PYTHONPATH}")
  endforeach()
endif()

# Install build system files and documentation
install(DIRECTORY cmake
  DESTINATION share/opm USE_SOURCE_PERMISSIONS
  PATTERN "OPM-CMake.md" EXCLUDE)

install(FILES cmake/OPM-CMake.md
  DESTINATION ${CMAKE_INSTALL_DOCDIR})

# Install tab completion skeleton
install(FILES etc/opm_bash_completion.sh.in DESTINATION share/opm/etc)

if (OPM_ENABLE_PYTHON)
  make_directory(${PROJECT_BINARY_DIR}/python)

  set(opm-common_PYTHON_PACKAGE_VERSION ${OPM_PYTHON_PACKAGE_VERSION_TAG})

  add_custom_target(copy_python ALL
    COMMAND ${Python3_EXECUTABLE} ${PROJECT_SOURCE_DIR}/python/install.py ${PROJECT_SOURCE_DIR}/python ${PROJECT_BINARY_DIR} 0)

  file(COPY ${PROJECT_SOURCE_DIR}/python/README.md ${PROJECT_SOURCE_DIR}/python/MANIFEST.in
       DESTINATION ${PROJECT_BINARY_DIR}/python)

  pybind11_add_module(opmcommon_python
                      ${PYTHON_CXX_SOURCE_FILES}
                      ${PROJECT_BINARY_DIR}/python/cxx/builtin_pybind11.cpp)

  target_link_libraries(opmcommon_python PRIVATE
                        opmcommon)
  if(TARGET pybind11::pybind11)
    target_link_libraries(opmcommon_python PRIVATE
                          pybind11::pybind11)
  else()
    target_include_directories(opmcommon_python SYSTEM PRIVATE ${pybind11_INCLUDE_DIRS})
  endif()
  set_target_properties(opmcommon_python PROPERTIES
                        LIBRARY_OUTPUT_DIRECTORY python/opm)
  add_dependencies(opmcommon_python copy_python)

  # Generate versioned setup.py
  configure_file(${PROJECT_SOURCE_DIR}/python/setup.py.in
                 ${PROJECT_BINARY_DIR}/python/setup.py.tmp)
  file(GENERATE OUTPUT ${PROJECT_BINARY_DIR}/python/setup.py
                INPUT ${PROJECT_BINARY_DIR}/python/setup.py.tmp)

  # Since the installation of Python code is nonstandard it is protected by an
  # extra cmake switch, OPM_INSTALL_PYTHON. If you prefer you can still invoke
  # setup.py install manually - optionally with the generated script
  # setup-install.sh - and completely bypass cmake in the installation phase.
  # If OPM_ENABLE_EMBEDDED_PYTHON is enabled, we also install opmcommon_python,
  # to make it available in a Python console and for e.g. opm-simulators.
  if (OPM_INSTALL_PYTHON OR OPM_ENABLE_EMBEDDED_PYTHON)
      include(PyInstallPrefix)
      install(TARGETS opmcommon_python DESTINATION ${DEST_PREFIX}${CMAKE_INSTALL_PREFIX}/${PYTHON_INSTALL_PREFIX}/opm)
      if (OPM_INSTALL_PYTHON)
        install(
          CODE "execute_process(
            COMMAND ${PYTHON_EXECUTABLE}
                    python/install.py
                    ${PROJECT_BINARY_DIR}/python/opm
                    ${DEST_PREFIX}${CMAKE_INSTALL_PREFIX}/${PYTHON_INSTALL_PREFIX} 1)")
      endif()
      if (OPM_ENABLE_EMBEDDED_PYTHON)
        install(
          CODE "execute_process(
            COMMAND ${PYTHON_EXECUTABLE}
                    python/install.py
                    ${PROJECT_BINARY_DIR}/python/opm_embedded
                    ${DEST_PREFIX}${CMAKE_INSTALL_PREFIX}/${PYTHON_INSTALL_PREFIX} 1)")
      endif()
      ## Need to install this Python script such that it can be used by opm-simulators when building against an installed
      ##  opm-common
      install( PROGRAMS "python/install.py" DESTINATION "${OPM_PYTHON_COMMON_DIR}" )
  endif()

  if (OPM_ENABLE_PYTHON OR OPM_INSTALL_PYTHON) # if OPM_ENABLE_EMBEDDED_PYTHON is true, then OPM_ENABLE_PYTHON is also automatically true
    ## Need to install this Python script such that it can be used by opm-simulators when building against an installed
    ##  opm-common
    install( PROGRAMS "python/generate_docstring_hpp.py" DESTINATION "${OPM_PYTHON_COMMON_DIR}" )
    install( FILES "python/docstrings_common.json" DESTINATION "${OPM_PYTHON_COMMON_DIR}" )
  endif()

  # Observe that if the opmcommon library has been built as a shared library the
  # python library opmcommon_python will in general not find it runtime while
  # testing.
  add_test(NAME python_tests
           WORKING_DIRECTORY ${PROJECT_BINARY_DIR}/python
           COMMAND ${CMAKE_COMMAND} -E env LD_LIBRARY_PATH=${PROJECT_BINARY_DIR}/lib ${Python3_EXECUTABLE} -m unittest discover
           )

  set_target_properties(opmcommon PROPERTIES POSITION_INDEPENDENT_CODE ON)
  set_directory_properties(PROPERTIES ADDITIONAL_MAKE_CLEAN_FILES ${PROJECT_BINARY_DIR}/python)

  # -------------------------------------------------------------------------
  # Let cmake configure some small shell scripts which can be used to simplify
  # building, testing and installation of the Python extensions.
  configure_file(python/setup-build.sh.in tmp/setup-build.sh)
  file( COPY ${PROJECT_BINARY_DIR}/tmp/setup-build.sh
        DESTINATION ${PROJECT_BINARY_DIR}
        FILE_PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE )

  configure_file(python/setup-test.sh.in tmp/setup-test.sh)
  file( COPY ${PROJECT_BINARY_DIR}/tmp/setup-test.sh
        DESTINATION ${PROJECT_BINARY_DIR}
        FILE_PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE )

  configure_file(python/setup-install.sh.in tmp/setup-install.sh)
  file( COPY ${PROJECT_BINARY_DIR}/tmp/setup-install.sh
        DESTINATION ${PROJECT_BINARY_DIR}
        FILE_PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE )

  configure_file(python/enable-python.sh.in enable-python.sh)

  # -------------------------------------------------------------------------
  # 2: Embed the Python interpreter for keywords like PYACTION and PYINPUT
  target_include_directories(opmcommon SYSTEM PRIVATE "${pybind11_INCLUDE_DIRS}")
  if (OPM_ENABLE_EMBEDDED_PYTHON)
    target_link_libraries(opmcommon PUBLIC ${PYTHON_LIBRARY})
    add_definitions(-DEMBEDDED_PYTHON)
  endif()
endif()

install(DIRECTORY docs/man1 DESTINATION ${CMAKE_INSTALL_MANDIR}
  FILES_MATCHING PATTERN "*.1")
