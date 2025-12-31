# Unit tests only: Modern Boost integration (CMake 3.30+)
# Use CMake's built-in Boost support instead of deprecated FindBoost module
# For CMake < 3.30, keep old behavior to avoid errors
if (CMAKE_VERSION VERSION_GREATER_EQUAL 3.30)
    cmake_policy(SET CMP0167 NEW)
endif ()

# Make CMake respect variables like BOOST_ROOT even though we look for "Boost"
if (CMAKE_VERSION VERSION_GREATER_EQUAL 3.27)
    cmake_policy(SET CMP0144 NEW)
endif ()

# Find Boost unit test framework. We don't want to link that to libftdi1 itself.
find_package(Boost COMPONENTS unit_test_framework REQUIRED)

enable_testing()

# CMake 3.28 compatibility: include_directories only needed for CMake < 3.30
# (imported targets handle include paths automatically in CMake 3.30+)
if (CMAKE_VERSION VERSION_LESS 3.30)
    INCLUDE_DIRECTORIES(BEFORE ${Boost_INCLUDE_DIRS})
endif ()

INCLUDE_DIRECTORIES(BEFORE ${PROJECT_SOURCE_DIR}/src)

set(cpp_tests basic.cpp baudrate.cpp)

add_executable(test_libftdi1 ${cpp_tests})

# Use modern imported targets for CMake 3.30+, fallback to old-style variables
if (CMAKE_VERSION VERSION_GREATER_EQUAL 3.30)
    target_link_libraries(test_libftdi1 ftdi1 Boost::unit_test_framework)
else ()
    target_link_libraries(test_libftdi1 ftdi1 ${Boost_UNIT_TEST_FRAMEWORK_LIBRARIES})
endif ()

add_test(test_libftdi1 test_libftdi1)

# Add custom target so we run easily run "make check"
add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND} DEPENDS test_libftdi1)
