cmake_minimum_required(VERSION 3.15 FATAL_ERROR)

message(STATUS "CMake version: ${CMAKE_VERSION}")

# Project
project(libftdi1 C)
set(MAJOR_VERSION 1)
set(MINOR_VERSION 5)
set(PACKAGE libftdi1)
set(VERSION_STRING ${MAJOR_VERSION}.${MINOR_VERSION})
set(VERSION ${VERSION_STRING})
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

# Target link properties INTERFACE_LINK_OPTIONS, INTERFACE_LINK_DIRECTORIES
# and INTERFACE_LINK_DEPENDS are now transitive over private dependencies
# of static libraries
if (POLICY CMP0099)
    cmake_policy(SET CMP0099 NEW)
endif ()

# CMake
if ("${CMAKE_BUILD_TYPE}" STREQUAL "")
    set(CMAKE_BUILD_TYPE RelWithDebInfo)
endif ("${CMAKE_BUILD_TYPE}" STREQUAL "")

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if (NOT PROJECT_SOURCE_DIR STREQUAL PROJECT_BINARY_DIR)
    # Auto-create a .gitignore in the out-of-source build directory.
    file(GENERATE OUTPUT .gitignore CONTENT "*")
endif ()

include(GNUInstallDirs)

add_definitions(-Wall)

include(CMakeOptions.txt)

# Debug build
message("-- Build type: ${CMAKE_BUILD_TYPE}")
if (${CMAKE_BUILD_TYPE} STREQUAL Debug)
    add_definitions(-DDEBUG)
endif (${CMAKE_BUILD_TYPE} STREQUAL Debug)

# Find libusb
find_package(LibUSB REQUIRED)
include_directories(${LIBUSB_INCLUDE_DIR})

# Find Boost
if (BUILD_TESTS)
    find_package(Boost REQUIRED)
endif ()

# Set components
set(CPACK_COMPONENTS_ALL sharedlibs staticlibs headers)
set(CPACK_COMPONENT_SHAREDLIBS_DISPLAY_NAME "Shared libraries")
set(CPACK_COMPONENT_STATICLIBS_DISPLAY_NAME "Static libraries")
set(CPACK_COMPONENT_HEADERS_DISPLAY_NAME "C++ Headers")

set(CPACK_COMPONENT_SHAREDLIBS_DESCRIPTION
        "Shared library for general use.")
set(CPACK_COMPONENT_STATICLIBS_DESCRIPTION
        "Static library, good if you want to embed libftdi1 in your application.")
set(CPACK_COMPONENT_HEADERS_DESCRIPTION
        "C/C++ header files.")

set(CPACK_COMPONENT_SHAREDLIBS_GROUP "Development")
set(CPACK_COMPONENT_STATICLIBS_GROUP "Development")
set(CPACK_COMPONENT_HEADERS_GROUP "Development")

# guess LIB_SUFFIX, don't take debian multiarch into account
if (NOT DEFINED LIB_SUFFIX)
    if (CMAKE_SYSTEM_NAME MATCHES "Linux"
            AND NOT CMAKE_CROSSCOMPILING
            AND NOT EXISTS "/etc/debian_version"
            AND NOT EXISTS "/etc/arch-release")
        if ("${CMAKE_SIZEOF_VOID_P}" EQUAL "8")
            set(LIB_SUFFIX 64)
        endif ()
    endif ()
endif ()

if (NOT APPLE)
    if (CMAKE_SIZEOF_VOID_P EQUAL 4)
        SET(PACK_ARCH "")
    elseif (CMAKE_SIZEOF_VOID_P EQUAL 8)
        SET(PACK_ARCH .x86_64)
    endif (CMAKE_SIZEOF_VOID_P EQUAL 4)
else (NOT APPLE)
    SET(PACK_ARCH "")
endif (NOT APPLE)

# Package information
set(CPACK_PACKAGE_VERSION ${VERSION_STRING})
set(CPACK_PACKAGE_CONTACT "Intra2net AG <libftdi@developer.intra2net.com>")
set(CPACK_PACKAGE_DESCRIPTION "libftdi1 library.")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY ${CPACK_PACKAGE_DESCRIPTION})

# Package settings
if (UNIX)
    set(CPACK_GENERATOR "DEB;RPM")
    set(CPACK_CMAKE_GENERATOR "Unix Makefiles")
    set(CPACK_PACKAGE_NAME ${PROJECT_NAME})
    set(CPACK_PACKAGE_FILE_NAME ${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}${PACK_ARCH})
endif ()

if (WIN32)
    set(CPACK_GENERATOR "NSIS")
    set(CPACK_CMAKE_GENERATOR "MinGW Makefiles")
    set(CPACK_PACKAGE_NAME "${PROJECT_NAME}")
    set(CPACK_PACKAGE_VENDOR "")
    set(CPACK_PACKAGE_INSTALL_DIRECTORY "libftdi1")
    set(CPACK_PACKAGE_FILE_NAME "${PROJECT_NAME}-${VERSION_STRING}-win32")
    set(CPACK_NSIS_DISPLAY_NAME "libftdi1")
    set(CPACK_NSIS_MODIFY_PATH ON)
endif ()

set(CPACK_RESOURCE_FILE_LICENSE ${CMAKE_CURRENT_SOURCE_DIR}/LICENSE)

set(CPACK_SOURCE_GENERATOR TGZ)
set(CPACK_SOURCE_IGNORE_FILES "\\\\.git;~$;build/")
set(CPACK_SOURCE_PACKAGE_FILE_NAME ${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION})

# Subdirectories
if (UNIX)
    set(CPACK_SET_DESTDIR ON)
endif ()

# "make dist" target
set(ARCHIVE_NAME ${CMAKE_PROJECT_NAME}-${VERSION_STRING})
add_custom_target(dist
        COMMAND git archive --prefix=${ARCHIVE_NAME}/ HEAD
        | bzip2 > ${PROJECT_BINARY_DIR}/${ARCHIVE_NAME}.tar.bz2
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})

add_subdirectory(src)
add_subdirectory(packages)

if (DOCUMENTATION)
    add_subdirectory(doc)
endif (DOCUMENTATION)

if (FTDIPP)
    project(libftdi1 C CXX)
    add_subdirectory(ftdipp)
endif (FTDIPP)

if (PYTHON_BINDINGS)
    add_subdirectory(python)
endif (PYTHON_BINDINGS)

if (FTDI_EEPROM)
    add_subdirectory(ftdi_eeprom)
endif (FTDI_EEPROM)

if (EXAMPLES)
    add_subdirectory(examples)
endif (EXAMPLES)

if (BUILD_TESTS)
    project(libftdi1 C CXX)
    add_subdirectory(test)
endif (BUILD_TESTS)

# PkgConfig
set(prefix ${CMAKE_INSTALL_PREFIX})
set(exec_prefix ${CMAKE_INSTALL_PREFIX}/bin)
set(includedir ${CMAKE_INSTALL_PREFIX}/include/${PROJECT_NAME})

if (${UNIX})
    set(libdir ${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR})
endif (${UNIX})
if (${WIN32})
    set(libdir ${CMAKE_INSTALL_PREFIX}/bin)
    if (STATICLIBS)
        list(APPEND libdir ${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR})
    endif ()
endif (${WIN32})

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libftdi1.pc.in ${CMAKE_CURRENT_BINARY_DIR}/libftdi1.pc @ONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libftdipp1.pc.in ${CMAKE_CURRENT_BINARY_DIR}/libftdipp1.pc @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libftdi1.pc ${CMAKE_CURRENT_BINARY_DIR}/libftdipp1.pc
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

if (UNIX OR MINGW)
    configure_file(libftdi1-config.in ${CMAKE_CURRENT_BINARY_DIR}/libftdi1-config @ONLY)
    install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/libftdi1-config
            DESTINATION bin)
endif ()

# config script install path
if (NOT DEFINED LIBFTDI_CMAKE_CONFIG_DIR)
    set(LIBFTDI_CMAKE_CONFIG_DIR ${CMAKE_INSTALL_LIBDIR}/cmake/libftdi1)
endif ()

set(LIBFTDI_INCLUDE_DIR ${includedir})
set(LIBFTDI_INCLUDE_DIRS ${LIBFTDI_INCLUDE_DIR})

# Set up library variables based on what's being built
if (SHAREDLIBS)
    set(LIBFTDI_LIBRARY ftdi1)
    set(LIBFTDI_LIBRARIES ${LIBFTDI_LIBRARY})
    list(APPEND LIBFTDI_LIBRARIES ${LIBUSB_LIBRARIES})
endif ()

if (STATICLIBS)
    set(LIBFTDI_STATIC_LIBRARY ftdi1.a)
    set(LIBFTDI_STATIC_LIBRARIES ${LIBFTDI_STATIC_LIBRARY})
    list(APPEND LIBFTDI_STATIC_LIBRARIES ${LIBUSB_LIBRARIES})
endif ()

if (FTDIPP)
    if (SHAREDLIBS)
        set(LIBFTDIPP_LIBRARY ftdipp1)
        set(LIBFTDIPP_LIBRARIES ${LIBFTDIPP_LIBRARY})
        list(APPEND LIBFTDIPP_LIBRARIES ${LIBUSB_LIBRARIES})
    endif ()
    if (STATICLIBS)
        set(LIBFTDIPP_STATIC_LIBRARY ftdipp1.a)
        set(LIBFTDIPP_STATIC_LIBRARIES ${LIBFTDIPP_STATIC_LIBRARY})
        list(APPEND LIBFTDIPP_STATIC_LIBRARIES ${LIBUSB_LIBRARIES})
    endif ()
endif ()

set(LIBFTDI_LIBRARY_DIRS ${libdir})
set(LIBFTDI_ROOT_DIR ${prefix})
set(LIBFTDI_VERSION_STRING ${VERSION_STRING})
set(LIBFTDI_VERSION_MAJOR ${MAJOR_VERSION})
set(LIBFTDI_VERSION_MINOR ${MINOR_VERSION})

set(LIBFTDI_USE_FILE ${CMAKE_INSTALL_PREFIX}/${LIBFTDI_CMAKE_CONFIG_DIR}/UseLibFTDI1.cmake)

include(CMakePackageConfigHelpers)

configure_package_config_file(
        cmake/LibFTDI1Config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/LibFTDI1Config.cmake
        INSTALL_DESTINATION ${LIBFTDI_CMAKE_CONFIG_DIR}
        PATH_VARS
        LIBFTDI_USE_FILE
        LIBFTDI_ROOT_DIR
        LIBFTDI_INCLUDE_DIR
        LIBFTDI_INCLUDE_DIRS
        LIBFTDI_LIBRARY_DIRS
        NO_CHECK_REQUIRED_COMPONENTS_MACRO
)
write_basic_package_version_file(
        LibFTDI1ConfigVersion.cmake
        VERSION ${LIBFTDI_VERSION_STRING}
        COMPATIBILITY AnyNewerVersion
)

install(
        FILES
        ${CMAKE_CURRENT_BINARY_DIR}/LibFTDI1Config.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/LibFTDI1ConfigVersion.cmake
        cmake/UseLibFTDI1.cmake
        DESTINATION ${LIBFTDI_CMAKE_CONFIG_DIR}
)

include(CPack)

message(STATUS "Summary of build options:

  Build shared libs: ${SHAREDLIBS}
  Build static libs: ${STATICLIBS}
  Build C++ bindings: ${FTDIPP}
  Build Python bindings: ${PYTHON_BINDINGS}
  Build ftdi_eeprom: ${FTDI_EEPROM}
  Build examples: ${EXAMPLES}
  Build tests: ${BUILD_TESTS}
  Build API documentation: ${DOCUMENTATION}
")
