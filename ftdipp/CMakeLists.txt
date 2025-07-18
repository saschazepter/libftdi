# Targets
set(cpp_sources ${CMAKE_CURRENT_SOURCE_DIR}/ftdi.cpp CACHE INTERNAL "List of cpp sources")
set(cpp_headers ${CMAKE_CURRENT_SOURCE_DIR}/ftdi.hpp CACHE INTERNAL "List of cpp headers")

# Includes
include_directories(BEFORE ${CMAKE_CURRENT_BINARY_DIR}
        ${CMAKE_CURRENT_SOURCE_DIR}
        ${PROJECT_SOURCE_DIR}/src
)

# Shared library
if (SHAREDLIBS)
    add_library(ftdipp1 SHARED ${cpp_sources})
    math(EXPR VERSION_FIXUP "${MAJOR_VERSION} + 1")    # Compatibility with previous releases
    set_target_properties(ftdipp1 PROPERTIES VERSION ${VERSION_FIXUP}.${MINOR_VERSION}.0 SOVERSION 3)
    # Prevent clobbering each other during the build
    set_target_properties(ftdipp1 PROPERTIES CLEAN_DIRECT_OUTPUT 1)
    # Dependencies
    target_link_libraries(ftdipp1 ftdi1 ${LIBUSB_LIBRARIES})
    install(TARGETS ftdipp1
            RUNTIME DESTINATION bin
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    )
endif ()

# Static library
if (STATICLIBS)
    add_library(ftdipp1-static STATIC ${cpp_sources})
    set_target_properties(ftdipp1-static PROPERTIES OUTPUT_NAME "ftdipp1")
    set_target_properties(ftdipp1-static PROPERTIES CLEAN_DIRECT_OUTPUT 1)
    target_link_libraries(ftdipp1-static ftdi1-static ${LIBUSB_LIBRARIES})
    install(TARGETS ftdipp1-static
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
            COMPONENT staticlibs
    )
endif ()

install(FILES ${cpp_headers}
        DESTINATION include/${PROJECT_NAME}
        COMPONENT headers
)
