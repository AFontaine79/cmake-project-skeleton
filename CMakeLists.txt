cmake_minimum_required(VERSION 3.17)
project(MYPROJECT
	VERSION 0.1
	DESCRIPTION "MYDESCRIPTION"
	LANGUAGES C CXX ASM)

set_property(GLOBAL PROPERTY C_STANDARD 11)
set_property(GLOBAL PROPERTY CXX_STANDARD 17)

include(cmake/CPM.cmake)

if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME OR PROJECTVARNAME_BUILD_TESTING)
    include(CTest)
endif()

include(cmake/compiler/CheckAndApplyFlags.cmake)
include(cmake/linker/AddExecutableWithLinkerScriptDep.cmake)
include(cmake/linker/Map.cmake)
include(cmake/Conversions.cmake)

include(BuildOptions.cmake)
include(Packaging.cmake)

#include(cmake/debugging/PrintTargetProperties.cmake)
#set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

##################
# Compiler Flags #
##################

# If we're not a subproject, globally apply our warning flags
if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
  add_compile_options(-Wall -Wextra)
endif()

if(ENABLE_COVERAGE)
  include(cmake/CodeCoverage.cmake)
  append_coverage_compiler_flags()
endif()

#####################
# CMocka Dependency #
#####################

if(PROJECTVARNAME_TESTING_IS_ENABLED)
  find_package(cmocka QUIET)
  if(NOT cmocka_FOUND)
    CPMAddPackage(
      NAME cmocka
      GIT_REPOSITORY https://git.cryptomilk.org/projects/cmocka.git/
      VERSION 1.1.5
      GIT_TAG cmocka-1.1.5
      DOWNLOAD_ONLY YES
    )

    # Convenience variable to shorten our code below
    set(CMOCKA_STATIC_FILENAME
      ${CMAKE_STATIC_LIBRARY_PREFIX}cmocka-static${CMAKE_STATIC_LIBRARY_SUFFIX}
    )

    # We would normally use CPMAddPackage, but CMocka's build presents problems.
    # So we include it as an external project, which allows us to build the lib
    # separately and then link it in.
    include(ExternalProject)
    ExternalProject_Add(project_cmocka
      SOURCE_DIR ${cmocka_SOURCE_DIR}
      PREFIX ${CMAKE_CURRENT_BINARY_DIR}/cmocka
      BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/cmocka
      CMAKE_ARGS
        -DBUILD_STATIC_LIB=ON
        -DWITH_STATIC_LIB=ON # Without this, Cmocka will not install the static lib
        -DWITH_EXAMPLES=OFF
        -DCMAKE_BUILD_TYPE=Debug
        -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/cmocka
      # This is needed with Ninja generators to prevent it from failing due to
      # the library being missing before the build has been run
      BUILD_BYPRODUCTS ${CMAKE_CURRENT_BINARY_DIR}/cmocka/lib/${CMOCKA_STATIC_FILENAME}
    )

    # We need to get the CMocka installation directory to know where our
    # built libraries can be found
    ExternalProject_Get_Property(project_cmocka BINARY_DIR)

    # Now we define an IMPORTED library target and tell CMake where the library
    # files can be found. To ensure the build flows in the proper order,
    # we'll add a dependency on the ExternalProject target above, which will
    # ensure the library is built before we try to link it
    add_library(cmocka-static STATIC IMPORTED)
    set_target_properties(cmocka-static PROPERTIES
      IMPORTED_LOCATION ${BINARY_DIR}/lib/${CMOCKA_STATIC_FILENAME}
    )
    add_dependencies(cmocka-static project_cmocka)

    # Maintain build compatibility between find_package and CMakeLists.txt variants.
    set(CMOCKA_LIBRARIES cmocka-static)
    set(CMOCKA_INCLUDE_DIR ${BINARY_DIR}/include)
  endif()
endif()

##########################
# Enable Static Analysis #
##########################

find_program(CPPCHECK cppcheck)

if(CPPCHECK)
  set(CPPCHECK_DEFAULT_ARGS
    ${CPPCHECK} --quiet --enable=style --force
      # Include directories
      -I ${CMAKE_CURRENT_LIST_DIR}/include
  )

  if(BUILD_WITH_STATIC_ANALYSIS)
    set(CMAKE_C_CPPCHECK ${CPPCHECK_DEFAULT_ARGS})
  endif()

  add_custom_target(cppcheck
    COMMAND ${CPPCHECK_DEFAULT_ARGS}
    # Source directories
    ${CMAKE_CURRENT_LIST_DIR}/src/ ${CMAKE_CURRENT_LIST_DIR}/target/
  )

  add_custom_target(cppcheck-xml
    COMMAND ${CPPCHECK_DEFAULT_ARGS}
    # enable XML output
    --xml --xml-version=2
    # Source directories
    ${CMAKE_CURRENT_LIST_DIR}/src/ ${CMAKE_CURRENT_LIST_DIR}/target/
    # Redirect to file
    2>${CMAKE_BINARY_DIR}/cppcheck.xml
  )
endif()

find_program(CLANG_TIDY clang-tidy)
if(CLANG_TIDY)
  if(BUILD_WITH_STATIC_ANALYSIS)
    set(CMAKE_C_CLANG_TIDY ${CLANG_TIDY})
  endif()

  add_custom_target(tidy
    COMMAND tools/clang-tidy.sh ${CMAKE_BINARY_DIR}
    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
  )
endif()

#######################
# Process Source Tree #
#######################

add_subdirectory(src)

#####################
# Process Test Tree #
#####################

add_subdirectory(test)

###################
# Tooling Targets #
###################

find_program(DOXYGEN doxygen)
if(DOXYGEN)
  add_custom_target(docs
    COMMAND ${DOXYGEN} docs/Doxyfile
    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
  )
  # This property will remove a directory, while add_custom_target BYPRODUCTS will not.
  set_target_properties(docs PROPERTIES ADDITIONAL_CLEAN_FILES ${CMAKE_BINARY_DIR}/doc/)
endif()

find_program(CLANG_FORMAT clang-format)
if(CLANG_FORMAT)
  add_custom_target(format
    COMMAND tools/format.sh
    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
  )

  add_custom_target(format-patch
    COMMAND tools/format-patch.sh
    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
  )
endif()

find_program(LIZARD lizard)
if(LIZARD)
  set(LIZARD_BASE_ARGS
      ${LIZARD}
      --length 75 # fail when functions longer than this
      --CCN 10 # fail over this CCN
      --arguments 10 # fail this arg count
  )
  set(LIZARD_PATHS ${CMAKE_CURRENT_LIST_DIR}/src ${CMAKE_CURRENT_LIST_DIR}/tests)

  add_custom_target(complexity
    COMMAND
      ${LIZARD_BASE_ARGS}
      -w # Only show warnings
      ${LIZARD_PATHS}
  )

  add_custom_target(complexity-full
    COMMAND
      ${LIZARD_BASE_ARGS}
      ${LIZARD_PATHS}
  )

  add_custom_target(complexity-xml
    COMMAND
      ${LIZARD_BASE_ARGS}
      --xml # Generate XML output
      ${LIZARD_PATHS}
      # Redirect output to file
      > ${CMAKE_BINARY_DIR}/complexity.xml
    BYPRODUCTS
      ${CMAKE_BINARY_DIR}/complexity.xml
  )
endif()

if(ENABLE_COVERAGE)
  setup_target_for_coverage_gcovr_xml(
    NAME coverage-xml
    EXECUTABLE ctest
    DEPENDENCIES libc_tests printf_tests
  )

  setup_target_for_coverage_gcovr_html(
    NAME coverage-html
    EXECUTABLE ctest
    DEPENDENCIES libc_tests printf_tests
  )

  add_custom_target(coverage
    DEPENDS coverage-xml coverage-html)
endif()
