# Includes
include_directories(BEFORE ${CMAKE_CURRENT_BINARY_DIR}
        ${CMAKE_CURRENT_SOURCE_DIR}
)

# Version information
set(SNAPSHOT_VERSION "unknown")
execute_process(COMMAND git describe
        OUTPUT_VARIABLE GIT_DESCRIBE_OUTPUT
        RESULT_VARIABLE GIT_DESCRIBE_RESULT
        OUTPUT_STRIP_TRAILING_WHITESPACE
)
if (${GIT_DESCRIBE_RESULT} STREQUAL 0)
    set(SNAPSHOT_VERSION ${GIT_DESCRIBE_OUTPUT})
endif ()
message(STATUS "Detected git snapshot version: ${SNAPSHOT_VERSION}")

configure_file(ftdi_version_i.h.in "${CMAKE_CURRENT_BINARY_DIR}/ftdi_version_i.h" @ONLY)

if (MSVC)
    # Disable some overly-verbose warnings activated by -Wall

    # '<name>': function not inlined
    add_compile_options(/wd4710)

    # function '<name>' selected for automatic inline expansion
    add_compile_options(/wd4711)

    # Compiler will insert Spectre mitigation for memory load if /Qspectre switch specified
    add_compile_options(/wd5045)

    # '<type>': '4' bytes padding added after data member '<name>'
    add_compile_options(/wd4820)

    # conversion from '<type1>' to '<type2>', possible loss of data
    add_compile_options(/wd4242 /wd4244 /wd4267)

    # Disable warning on unsafe string functions
    add_compile_definitions(_CRT_SECURE_NO_WARNINGS)

    if (MSVC_VERSION GREATER_EQUAL 1913)
        # Disable warnings from system headers
        add_compile_options(/external:anglebrackets /external:W0)
    endif ()
endif ()

# Targets
set(c_sources ${CMAKE_CURRENT_SOURCE_DIR}/ftdi.c ${CMAKE_CURRENT_SOURCE_DIR}/ftdi_stream.c CACHE INTERNAL "List of c sources")
set(c_headers ${CMAKE_CURRENT_SOURCE_DIR}/ftdi.h CACHE INTERNAL "List of c headers")

# Shared library
if (SHAREDLIBS)
    add_library(ftdi1 SHARED ${c_sources})
    math(EXPR VERSION_FIXUP "${MAJOR_VERSION} + 1")    # Compatibility with previous releases
    set_target_properties(ftdi1 PROPERTIES VERSION ${VERSION_FIXUP}.${MINOR_VERSION}.0 SOVERSION 2)
    # Prevent clobbering each other during the build
    set_target_properties(ftdi1 PROPERTIES CLEAN_DIRECT_OUTPUT 1)
    target_link_libraries(ftdi1 ${LIBUSB_LIBRARIES})
    install(TARGETS ftdi1
            RUNTIME DESTINATION bin
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    )
endif ()

# Static library
if (STATICLIBS)
    add_library(ftdi1-static STATIC ${c_sources})
    target_link_libraries(ftdi1-static ${LIBUSB_LIBRARIES})
    set_target_properties(ftdi1-static PROPERTIES OUTPUT_NAME "ftdi1")
    set_target_properties(ftdi1-static PROPERTIES CLEAN_DIRECT_OUTPUT 1)
    install(TARGETS ftdi1-static
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
            COMPONENT staticlibs
    )
endif ()

install(FILES ${c_headers}
        DESTINATION include/${PROJECT_NAME}
        COMPONENT headers
)
