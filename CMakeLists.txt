cmake_minimum_required(VERSION 3.2.2)
project(FACTER VERSION 3.14.9)

# Set this early, so it's available. AIX gets weird, man.
if("${CMAKE_SYSTEM_NAME}" MATCHES "AIX")
    set(AIX TRUE)
endif()

if (NOT CMAKE_BUILD_TYPE)
    message(STATUS "Defaulting to a release build.")
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif()

include(CheckIncludeFile)
CHECK_INCLUDE_FILE(utmpx.h HAVE_UTMPX_H -DHAVE_UTMPX_H)

option(YAMLCPP_STATIC "Use yaml-cpp's static libraries" OFF)
option(BUILD_SHARED_LIBS "Build libfacter as a shared library" ON)

set(FACTER_PATH "" CACHE PATH "Specify the location to look for specific binaries before trying PATH.")
if (FACTER_PATH)
    # Specify a preferred location for binary lookup that will be prioritized over PATH.
    file(TO_CMAKE_PATH ${FACTER_PATH} FACTER_PATH_FIXED)
    add_definitions(-DFACTER_PATH="${FACTER_PATH_FIXED}")
    message(STATUS "Prioritizing binary lookup in ${FACTER_PATH_FIXED}")
endif()

set(FACTER_RUBY "" CACHE FILEPATH "Specify the location of libruby at compile-time, bypassing dynamic lookup.")
if (FACTER_RUBY)
    file(TO_CMAKE_PATH ${FACTER_RUBY} FACTER_RUBY_PATH)
    add_definitions(-DFACTER_RUBY="${FACTER_RUBY_PATH}")
    message(STATUS "Fixing lookup for libruby to ${FACTER_RUBY_PATH}")
endif()

set(AIO_AGENT_VERSION "" CACHE STRING "Creates an aio_agent_version fact with the specified value.")
if (AIO_AGENT_VERSION)
    add_definitions(-DAIO_AGENT_VERSION="${AIO_AGENT_VERSION}")
    message(STATUS "Adding fact aio_agent_version=${AIO_AGENT_VERSION}")
endif()

enable_testing()

list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake")

if ("${CMAKE_SYSTEM_NAME}" MATCHES "Darwin")
    # Allow searching in boxen installed homebrew directories
    # http://stackoverflow.com/questions/1487752/how-do-i-instruct-cmake-to-look-for-libraries-installed-by-macports
    set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} /opt/boxen/homebrew/lib)
    set(CMAKE_INCLUDE_PATH ${CMAKE_INCLUDE_PATH} /opt/boxen/homebrew/include)
endif()

include(FeatureSummary)

SET(LEATHERMAN_COMPONENTS locale catch nowide logging util file_util dynamic_library execution ruby rapidjson)

# We look for curl early, because whether or not we link to the leatherman curl library
# is dependant on whether or not we find curl on the system.
if ((("${CMAKE_SYSTEM_NAME}" MATCHES "FreeBSD|Linux|OpenBSD") OR WIN32) AND NOT WITHOUT_CURL)
    find_package(CURL)
    if (CURL_FOUND)
        add_definitions(-DUSE_CURL)
        list(APPEND LEATHERMAN_COMPONENTS curl)
    endif()
    set_package_properties(CURL PROPERTIES DESCRIPTION "A free and easy-to-use client-side URL transfer library" URL "http://curl.haxx.se/libcurl/")
    set_package_properties(CURL PROPERTIES TYPE OPTIONAL PURPOSE "Enables facts that require HTTP.")
endif()

if (WIN32)
    list(APPEND LEATHERMAN_COMPONENTS windows)
endif()

find_package(Leatherman REQUIRED COMPONENTS ${LEATHERMAN_COMPONENTS})

# Now that we have leatherman, we can pulll in its options file, which
# we need for finding all our other libraries.
include(options)

if (LEATHERMAN_USE_LOCALES)
    add_definitions(-DLEATHERMAN_I18N)
    SET(BOOST_COMPONENTS locale)
endif()

# We use program_options, system, filesystem, date_time, and regex directly. Testing uses thread and chrono.
list(APPEND BOOST_COMPONENTS program_options system filesystem date_time regex thread chrono)

find_package(Boost 1.54 REQUIRED COMPONENTS ${BOOST_COMPONENTS})
# date_time and regex need threads on some platforms, and find_package Boost only includes
# pthreads if you require the Boost.Thread component.
find_package(Threads)

find_package(Ruby 1.9)

find_package(YAMLCPP REQUIRED)
if (NOT WITHOUT_OPENSSL)
    find_package(OPENSSL)
endif()
if ("${CMAKE_SYSTEM_NAME}" MATCHES "Linux" AND NOT WITHOUT_BLKID)
    find_package(BLKID)
endif()

find_package(CPPHOCON REQUIRED)

if ("${CMAKE_SYSTEM_NAME}" MATCHES "Linux")
    find_package(UDEV)
endif()

find_package(WHEREAMI)
if (WHEREAMI_FOUND)
    add_definitions(-DUSE_WHEREAMI)
endif()

if (NOT WITHOUT_JRUBY AND NOT WIN32)
    find_package(JNI)
    set_package_properties(JNI PROPERTIES DESCRIPTION "Java Native Interface (JNI) is a programming framework that enables Java code running in a Java Virtual Machine (JVM) to call and be called by native applications.")
    set_package_properties(JNI PROPERTIES TYPE OPTIONAL PURPOSE "Enables JRuby support in Facter.")

    if (JNI_FOUND)
        find_package(Java)
        set_package_properties(Java PROPERTIES DESCRIPTION "Java compiler for JNI.")
        set_package_properties(Java PROPERTIES TYPE OPTIONAL PURPOSE "Enables JRuby support in Facter.")

        if (Java_JAVAC_EXECUTABLE)
            set(JRUBY_SUPPORT TRUE)
            set(CMAKE_JAVA_COMPILE_FLAGS -source 1.6 -target 1.6)
            add_definitions(-DUSE_JRUBY_SUPPORT)
        endif()
    endif()
endif()

# Display a summary of the features
include(FeatureSummary)
feature_summary(WHAT ALL)

# Set RPATH if not installing to a system library directory
list(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/lib" INSTALL_IS_SYSTEM_DIR)
if ("${INSTALL_IS_SYSTEM_DIR}" STREQUAL "-1")
    set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
endif()

# Pull in common cflags setting from leatherman
include(cflags)
set(FACTER_CXX_FLAGS "${LEATHERMAN_CXX_FLAGS}")
add_definitions(${LEATHERMAN_DEFINITIONS})

option(DYNAMICBASE "Add dynamicbase linker option" ON)
# Enable DEP/ASLR switches on windows versions
if (WIN32)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--nxcompat")
      if (DYNAMICBASE)
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--dynamicbase")
      endif()
endif()

#
# Add cpplint and cppcheck targets
#
file(GLOB_RECURSE ALL_SOURCES lib/src/*.cc lib/inc/*.hpp lib/inc/*.h exe/*.cc exe/*.hpp exe/*.h)
add_cpplint_files(${ALL_SOURCES})
enable_cpplint()

add_cppcheck_dirs("${PROJECT_SOURCE_DIR}/lib" "${PROJECT_SOURCE_DIR}/exe")
enable_cppcheck()

# Pull in helper macros for working with leatherman libraries
include(leatherman)

set(CMAKE_REQUIRED_LIBRARIES ${LEATHERMAN_LIBRARIES})
CHECK_CXX_SOURCE_COMPILES("
#include <leatherman/util/environment.hpp>
int main() {
    int a = leatherman::util::environment::get_int(\"A\", 1000);
    return 0;
}
" HAS_LTH_GET_INT)

if (HAS_LTH_GET_INT)
    add_definitions(-DHAS_LTH_GET_INT)
endif()

    # Guard to make sure we don't compile what we don't have in Leatherman
if (WIN32)
    set(CMAKE_REQUIRED_INCLUDES ${LEATHERMAN_INCLUDE_DIRS})
    set(CMAKE_REQUIRED_FLAGS "-std=c++11")
    CHECK_CXX_SOURCE_COMPILES("
    #include \"leatherman/windows/registry.hpp\"

    using namespace leatherman::windows;
    int main() {
		int test = 0;
        try {
            test = registry::get_registry_dword(registry::HKEY::LOCAL_MACHINE, \"RandomSubkey\", \"RandomValue\");
        } catch (registry_exception &e) {
            return test;
        }
    }
    " HAS_LTH_GET_DWORD)

    if (HAS_LTH_GET_DWORD)
        add_definitions(-DHAS_LTH_GET_DWORD)
    endif()
endif()

add_subdirectory(lib)
add_subdirectory(exe)
add_subdirectory(locales)

# Add test executables for unit testing
add_test(NAME "libfacter\\ tests" COMMAND libfacter_test)
if (RUBY_FOUND)
    find_program(BUNDLER_PATH NAMES bundle.bat bundle)
    if (BUNDLER_PATH)
        message(STATUS "Bundler found, installing dependencies for Ruby tests...")
        execute_process(COMMAND ${BUNDLER_PATH} install --retry 2 --path=vendor/bundle WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/lib")
        add_test(NAME "libfacter\\ specs" COMMAND ${BUNDLER_PATH} exec rspec WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/lib")
    endif()
endif()
add_test(NAME "facter\\ smoke" COMMAND facter)

# Install the man page
if ("${CMAKE_SYSTEM_NAME}" MATCHES "OpenBSD|FreeBSD")
    set(MANDIR man/man8/)
else()
    set(MANDIR share/man/man8/)
endif()

option(INSTALL_BATCH_FILES "Generate .bat files during installation" ON)
install(FILES ${PROJECT_SOURCE_DIR}/man/man8/facter.8 DESTINATION ${MANDIR})
if (WIN32 AND INSTALL_BATCH_FILES)
  install(FILES ${PROJECT_SOURCE_DIR}/ext/windows/facter.bat DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)
  install(FILES ${PROJECT_SOURCE_DIR}/ext/windows/facter_interactive.bat DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)
  install(FILES ${PROJECT_SOURCE_DIR}/ext/windows/run_facter_interactive.bat DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)
endif()
