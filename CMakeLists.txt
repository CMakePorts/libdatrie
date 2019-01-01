cmake_minimum_required(VERSION 3.11)

##########################
#  Project configuration #
##########################

project(libdatrie LANGUAGES C)
include(GenerateExportHeader)
include(CMakePackageConfigHelpers)

set(LIBDATRIE_MAJOR_VERSION 0)
set(LIBDATRIE_MINOR_VERSION 2)
set(LIBDATRIE_PATCH_VERSION 12)
set(VERSION ${LIBDATRIE_MAJOR_VERSION}.${LIBDATRIE_MINOR_VERSION}.${LIBDATRIE_PATCH_VERSION})

option(BUILD_SHARED_LIBS "Build shared libs" OFF)
option(BUILD_TESTS "Build test" OFF)
option(BUILD_TOOLS "Build tools" ON)

##########################
#        libdatrie       #
##########################

set(LIBDATRIE_HEADERS 
  "datrie/typedefs.h"
  "datrie/triedefs.h"
  "datrie/alpha-map.h"
  "datrie/trie.h"
)

set(LIBDATRIE_SOURCES 
  "datrie/alpha-map.c"
  "datrie/darray.c"
  "datrie/dstring.c"
  "datrie/fileutils.c"
  "datrie/tail.c"
  "datrie/trie.c"
  "datrie/trie-string.c"
)

if(WIN32)
  set(LIBDATRIE_RESOURCES 
    "datrie/libdatrie.def"
  )
else()
  set(LIBDATRIE_RESOURCES
    "datrie/libdatrie.map"
  )
endif()

add_library(libdatrie "${LIBDATRIE_HEADERS}" "${LIBDATRIE_SOURCES}" "${LIBDATRIE_RESOURCES}")

target_include_directories(libdatrie PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  $<INSTALL_INTERFACE:include>
)

target_include_directories(libdatrie PRIVATE
  "datrie"
)

set_target_properties(libdatrie PROPERTIES PREFIX "")
if(WIN32)
  set_target_properties(libdatrie PROPERTIES DEBUG_POSTFIX "d")
endif()

if(MSVC)
  target_compile_definitions(libdatrie PRIVATE "-D_CRT_SECURE_NO_WARNINGS")
endif()

#Cmake config generation
configure_package_config_file(
  "cmake/libdatrie-config.cmake.in"
  "libdatrie-config.cmake"
  INSTALL_DESTINATION "lib/libdatrie"
)

write_basic_package_version_file(
  "libdatrie-config-version.cmake"
  VERSION ${VERSION}
  COMPATIBILITY AnyNewerVersion)

install(
  FILES "${CMAKE_CURRENT_BINARY_DIR}/libdatrie-config.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/libdatrie-config-version.cmake"
  DESTINATION "lib/libdatrie"
)


##########################
#         Tools          #
##########################

if(BUILD_TOOLS)

  add_executable(trietool "tools/trietool.c" )
  target_include_directories(trietool PRIVATE ".")
  target_link_libraries(trietool libdatrie)

  #suppress warnings
  if(MSVC)
    target_compile_definitions(trietool PRIVATE "-D_CRT_SECURE_NO_WARNINGS")
  endif()

  #find iconv
  find_package(Iconv REQUIRED)
  target_link_libraries(trietool Iconv::Iconv)

  #make configure file
  list(APPEND CMAKE_REQUIRED_LIBRARIES ${Iconv_LIBRARIES})
  include(CheckFunctionExists)
  check_function_exists(nl_langinfo HAVE_LANGINFO_CODESET)
  check_function_exists(locale_charset HAVE_LOCALE_CHARSET)
  configure_file(cmake/config.h.in config.h)
  target_include_directories(trietool PRIVATE ${CMAKE_CURRENT_BINARY_DIR})

endif()


##########################
#         Tests          #
##########################

if(BUILD_TESTS)
  enable_testing()
  
  set(TESTS_LIST
    "test_byte_alpha"
    "test_byte_list"
    "test_file"
    "test_iterator"
    "test_nonalpha"
    "test_null_trie"
    "test_store-retrieve"
    "test_term_state"
    "test_walk"
  )

  foreach(testname ${TESTS_LIST})
    add_executable(${testname} "tests/${testname}.c" "tests/utils.c")
    target_include_directories(${testname} PRIVATE "tests")
    target_link_libraries(${testname} libdatrie)
    target_compile_definitions(${testname} PRIVATE "-D_CRT_NONSTDC_NO_WARNINGS")
    add_test(${testname} ${testname})
  endforeach()
endif()

##########################
#      Installation      #
##########################

install(
  TARGETS libdatrie
  EXPORT libdatrie-targets
  RUNTIME DESTINATION "bin"
  ARCHIVE DESTINATION "lib"
  LIBRARY DESTINATION "lib"
)

install(
  EXPORT libdatrie-targets 
  NAMESPACE libdatrie::
  DESTINATION "lib/libdatrie"
)

install(
  FILES ${LIBDATRIE_HEADERS}
  DESTINATION "include/datrie"
)

if(BUILD_TOOLS)

  install(
    TARGETS trietool
    RUNTIME DESTINATION bin
  )

endif()
