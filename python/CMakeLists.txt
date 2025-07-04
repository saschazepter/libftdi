# workaround for cmake bug #0013449
if (NOT DEFINED CMAKE_FIND_ROOT_PATH OR NOT CMAKE_VERSION VERSION_LESS 3.0.0)
    find_package(SWIG REQUIRED)
else ()
    find_program(SWIG_EXECUTABLE NAMES swig2.0 swig)
    if (SWIG_EXECUTABLE)
        set(SWIG_USE_FILE ${CMAKE_ROOT}/Modules/UseSWIG.cmake)
        set(SWIG_FOUND TRUE)
    endif ()
endif ()
find_package(Python3 COMPONENTS Interpreter Development)

include(UseSWIG)
include_directories(BEFORE ${PROJECT_SOURCE_DIR}/src)
include_directories(${Python3_INCLUDE_DIRS})
link_directories(${CMAKE_CURRENT_BINARY_DIR}/../src)

if (DOCUMENTATION)
    set(CMAKE_SWIG_FLAGS -DDOXYGEN=${DOXYGEN_FOUND})
    # manually add dependency for new cmake / swig versions
    set_property(SOURCE ftdi1.i PROPERTY DEPENDS doc_i)
endif ()
swig_add_library(pyftdi1 LANGUAGE python SOURCES ftdi1.i)
swig_link_libraries(pyftdi1 ftdi1)

if (LINK_PYTHON_LIBRARY)
    swig_link_libraries(pyftdi1 ${PYTHON_LIBRARIES})
elseif (APPLE)
    set_target_properties(pyftdi1 PROPERTIES LINK_FLAGS "-undefined dynamic_lookup")
endif ()

set_target_properties(pyftdi1 PROPERTIES NO_SONAME ON)

execute_process(COMMAND ${Python3_EXECUTABLE} -c "import sysconfig; print( sysconfig.get_path( 'platlib', vars={'platbase': '${CMAKE_INSTALL_PREFIX}'} ) )"
        OUTPUT_VARIABLE _ABS_PYTHON_MODULE_PATH
        OUTPUT_STRIP_TRAILING_WHITESPACE)

get_filename_component(_ABS_PYTHON_MODULE_PATH ${_ABS_PYTHON_MODULE_PATH} ABSOLUTE)
file(RELATIVE_PATH _REL_PYTHON_MODULE_PATH ${CMAKE_INSTALL_PREFIX} ${_ABS_PYTHON_MODULE_PATH})

set(PYTHON_MODULE_PATH ${_REL_PYTHON_MODULE_PATH})

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/ftdi1.py DESTINATION ${PYTHON_MODULE_PATH})
install(TARGETS pyftdi1 LIBRARY DESTINATION ${PYTHON_MODULE_PATH})

if (DOCUMENTATION)
    # Run doxygen to only generate the xml
    add_custom_command(OUTPUT ${PROJECT_BINARY_DIR}/doc/xml/ftdi_8c.xml
            COMMAND ${DOXYGEN_EXECUTABLE} ${PROJECT_BINARY_DIR}/doc/Doxyfile.xml
            WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
            COMMENT "Generating ftdi_8c.xml"
            DEPENDS ${c_headers};${c_sources};${cpp_sources};${cpp_headers}
    )

    # generate .i from doxygen .xml
    add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/ftdi1_doc.i
            COMMAND ${Python3_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/doxy2swig.py -n
            ${PROJECT_BINARY_DIR}/doc/xml/ftdi_8c.xml
            ${CMAKE_CURRENT_BINARY_DIR}/ftdi1_doc.i
            COMMENT "Generating ftdi1_doc.i from ftdi_8c.xml"
            DEPENDS ${PROJECT_BINARY_DIR}/doc/xml/ftdi_8c.xml
    )
    add_custom_target(doc_i
            COMMENT "Python API bindings documentation"
            DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/ftdi1_doc.i
    )
    add_dependencies(pyftdi1 doc_i)

endif (DOCUMENTATION)

set(LIBFTDI_PYTHON_MODULE_PATH ${CMAKE_INSTALL_PREFIX}/${PYTHON_MODULE_PATH})
set(LIBFTDI_PYTHON_MODULE_PATH ${LIBFTDI_PYTHON_MODULE_PATH} PARENT_SCOPE) # for ftdiconfig.cmake

add_subdirectory(examples)
