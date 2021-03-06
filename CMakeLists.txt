# Set minimum CMake version
CMAKE_MINIMUM_REQUIRED(VERSION 2.8)


# Silence CMake warnings by adopting modern behavior for MACOSX_RPATH on newer
# versions of CMake
IF(POLICY CMP0042)
    CMAKE_POLICY(SET CMP0042 NEW)
ENDIF()


# Create the project
PROJECT(keytar)


# Set up options
OPTION(BUILD_SHARED_LIBS "Build shared libraries (use OFF for static)" ON)
OPTION(BUILD_TESTS "Enable test building" ON)
SET(BIN_INSTALL_DIR bin CACHE PATH "Install subdirectory for binaries")
SET(LIB_INSTALL_DIR lib CACHE PATH "Install subdirectory for libraries")
SET(INCLUDE_INSTALL_DIR include CACHE PATH "Install subdirectory for headers")


# Add local include directories
INCLUDE_DIRECTORIES(src)


# Handle common configuration and configuration based on platform
SET(KEYTAR_LINK_LIBRARIES)
SET(KEYTAR_SOURCES
    src/keytar_common.cc
)
IF(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    # Windows doesn't require any special include paths or libraries

    # Add source files
    LIST(APPEND KEYTAR_SOURCES src/keytar_win.cc)
ELSEIF(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    # On OS X, we need to link with the CoreFoundation and Security frameworks
    FIND_LIBRARY(CORE_FOUNDATION_FRAMEWORK CoreFoundation)
    IF(NOT CORE_FOUNDATION_FRAMEWORK)
        MESSAGE(FATAL_ERROR "Unable to locate CoreFoundation framework")
    ENDIF()
    FIND_LIBRARY(SECURITY_FRAMEWORK Security)
    IF(NOT SECURITY_FRAMEWORK)
        MESSAGE(FATAL_ERROR "Unable to locate Security framework")
    ENDIF()
    LIST(APPEND KEYTAR_LINK_LIBRARIES
        ${CORE_FOUNDATION_FRAMEWORK}
        ${SECURITY_FRAMEWORK}
    )

    # Add source files
    LIST(APPEND KEYTAR_SOURCES src/keytar_mac.cc)
ELSE()
    # On other POSIX systems, the only currently supported backend is GNOME
    # Keyring
    INCLUDE(FindPkgConfig)
    PKG_CHECK_MODULES(GNOME_KEYRING REQUIRED gnome-keyring-1)
    INCLUDE_DIRECTORIES(${GNOME_KEYRING_INCLUDE_DIRS})
    LIST(APPEND KEYTAR_LINK_LIBRARIES ${GNOME_KEYRING_LIBRARIES})

    # Add source files
    LIST(APPEND KEYTAR_SOURCES src/keytar_posix.cc)
ENDIF()


# Create library (allow user to specify type)
ADD_LIBRARY(keytar ${KEYTAR_SOURCES})
TARGET_LINK_LIBRARIES(keytar ${KEYTAR_LINK_LIBRARIES})


# Create an install target
INSTALL(TARGETS keytar
    RUNTIME DESTINATION "${BIN_INSTALL_DIR}"
    LIBRARY DESTINATION "${LIB_INSTALL_DIR}"
    ARCHIVE DESTINATION "${LIB_INSTALL_DIR}"
)
INSTALL(FILES src/keytar.h DESTINATION "${INCLUDE_INSTALL_DIR}")


# Add tests if request
IF(BUILD_TESTS)
    # Enable testing
    ENABLE_TESTING()

    # Create the test executable
    ADD_EXECUTABLE(keytar_test test/keytar_test.cc)
    TARGET_LINK_LIBRARIES(keytar_test keytar)

    # Add the test
    ADD_TEST(keytar_test keytar_test)
ENDIF()
