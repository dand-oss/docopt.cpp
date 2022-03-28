cmake_minimum_required(VERSION 3.1)
project(docopt.cpp VERSION 0.6.2)

###############
# ASI default settings
list(APPEND CMAKE_MODULE_PATH "$ENV{ASV_CMAKE}")
include(asv_cmake_defaults)
asv_cmake_defaults( )
###############

include(GNUInstallDirs)

#============================================================================
# Settable options
#============================================================================
option(WITH_TESTS "Build tests." OFF)
option(WITH_EXAMPLE "Build example." OFF)
option(USE_BOOST_REGEX "Replace std::regex with Boost.Regex" ON)

#============================================================================
# Internal compiler options
#============================================================================
# C++ standard
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
if(NOT CMAKE_CXX_STANDARD OR CMAKE_CXX_STANDARD LESS 11)
	set(CMAKE_CXX_STANDARD 11)
endif()

#============================================================================
# Sources & headers
#============================================================================
set(docopt_SOURCES docopt.cpp)
set(docopt_HEADERS
		docopt.h
		docopt_private.h
		docopt_util.h
		docopt_value.h
		)

#============================================================================
# Compile targets
#============================================================================
if(XCODE)
    # MSVC requires __declspec() attributes, which are achieved via the 
    # DOCOPT_DLL and DOCOPT_EXPORTS macros below. Since those macros are only
    # defined when building a shared library, we must build the shared and
    # static libraries completely separately.
    # Xcode does not support libraries with only object files as sources.
    # See https://cmake.org/cmake/help/v3.0/command/add_library.html?highlight=add_library
    add_library(docopt SHARED ${docopt_SOURCES} ${docopt_HEADERS})
    add_library(docopt_s STATIC ${docopt_SOURCES} ${docopt_HEADERS})
else()
    # If not using Xcode, we will create an intermediate object target
    # to avoid compiling the source code twice.
    add_library(docopt_o OBJECT ${docopt_SOURCES} ${docopt_HEADERS})
    set_target_properties(docopt_o PROPERTIES POSITION_INDEPENDENT_CODE TRUE)

    add_library(docopt SHARED $<TARGET_OBJECTS:docopt_o>)
	set_target_properties(docopt PROPERTIES
			VERSION ${PROJECT_VERSION}
			SOVERSION ${PROJECT_VERSION_MAJOR}
			)
    set_target_properties(docopt PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS ON)
    add_library(docopt_s STATIC $<TARGET_OBJECTS:docopt_o>)
endif()

target_include_directories(docopt PUBLIC $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}> $<INSTALL_INTERFACE:include/docopt>)
target_include_directories(docopt_s PUBLIC $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}> $<INSTALL_INTERFACE:include/docopt>)

if(NOT MSVC)
	set_target_properties(docopt PROPERTIES OUTPUT_NAME docopt)
	set_target_properties(docopt_s PROPERTIES OUTPUT_NAME docopt)
endif()

if(USE_BOOST_REGEX)
    add_definitions("-DDOCTOPT_USE_BOOST_REGEX")

    # find boost
    ##Set(Boost_DEBUG ON)
    if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
        Set(Boost_COMPILER "-clang40") # doh!
    endif()
    if(NOT BOOST_ROOT)
      set(BOOST_ROOT $ENV{BOOST_ROOT} CACHE PATH "Boost Root Directory")
    endif(NOT BOOST_ROOT)
    set(Boost_NO_SYSTEM_PATHS ON)
    if(USE_STATIC_BOOST)
        # static boost on windows add "lib" prefix
        # thus can coexist with dll version
        set(Boost_USE_STATIC_LIBS ON)
    else()
        set(Boost_USE_STATIC_LIBS OFF)
        add_definitions("-DBOOST_ALL_DYN_LINK")
    endif()
    set(Boost_USE_MULTITHREADED ON)
    # turn on __declspec(dllimport) modifiers
    # Find boost in "module" mode, not CONFIG mode - not BoostConfig.cmake
    find_package(Boost 1.78.0 EXACT REQUIRED COMPONENTS
        regex
        )

    if (NOT XCODE)
        target_include_directories( docopt_o PRIVATE ${Boost_INCLUDE_DIRS})
    endif (NOT XCODE)
    target_link_libraries(docopt PUBLIC Boost::regex)
	if(WITH_STATIC)
        target_link_libraries(docopt_s PRIVATE Boost::regex)
	endif()
endif()

#============================================================================
# Examples
#============================================================================
if(WITH_EXAMPLE)
	add_executable(docopt_example examples/naval_fate.cpp)
	target_link_libraries(docopt_example docopt)
endif()

#============================================================================
# Tests
#============================================================================
if(WITH_TESTS)
	set(TESTPROG "${CMAKE_CURRENT_BINARY_DIR}/run_testcase")
	set(TESTCASES "${PROJECT_SOURCE_DIR}/testcases.docopt")
	add_executable(run_testcase run_testcase.cpp)
	target_link_libraries(run_testcase docopt)
	configure_file(
			"${PROJECT_SOURCE_DIR}/run_tests.py"
			"${CMAKE_CURRENT_BINARY_DIR}/run_tests"
			ESCAPE_QUOTES
	)
	add_test("Testcases docopt" ${TESTPROG})
endif()

#============================================================================
# Install
#============================================================================
set(export_name "docopt-targets")

# Runtime package
install(TARGETS docopt EXPORT ${export_name} DESTINATION ${CMAKE_INSTALL_LIBDIR})

# Development package
install(TARGETS docopt_s EXPORT ${export_name} DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES ${docopt_HEADERS} DESTINATION include/docopt)

# CMake Package
include(CMakePackageConfigHelpers)
write_basic_package_version_file("${PROJECT_BINARY_DIR}/docopt-config-version.cmake" COMPATIBILITY SameMajorVersion)
install(FILES docopt-config.cmake ${PROJECT_BINARY_DIR}/docopt-config-version.cmake DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/docopt")
install(EXPORT ${export_name} DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/docopt")

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/docopt.pc.in ${CMAKE_CURRENT_BINARY_DIR}/docopt.pc @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/docopt.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

#============================================================================
# CPack
#============================================================================
set(CPACK_PACKAGE_NAME "docopt")
set(CPACK_DEBIAN_PACKAGE_DEPENDS "")
set(CPACK_RPM_PACKAGE_REQUIRES "")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Beautiful command line interfaces")
set(CPACK_PACKAGE_VENDOR "Jared Grubb")
set(CPACK_PACKAGE_CONTACT ${CPACK_PACKAGE_VENDOR})
set(CPACK_PACKAGE_DESCRIPTION_FILE "${PROJECT_SOURCE_DIR}/README.rst")
set(CPACK_RESOURCE_FILE_LICENSE "${PROJECT_SOURCE_DIR}/LICENSE-MIT")
set(CPACK_PACKAGE_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PROJECT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PROJECT_VERSION_PATCH})
set(CPACK_DEBIAN_PACKAGE_SECTION "Development")
set(CPACK_RPM_PACKAGE_GROUP "Development/Libraries")
set(CPACK_RPM_PACKAGE_LICENSE "MIT")
set(CPACK_STRIP_FILES TRUE)
include(CPack)
