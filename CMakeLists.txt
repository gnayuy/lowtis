CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
project (lowtis)

include (ExternalProject)

set (RUN_ENVIRONMENT "Workstation" CACHE TYPE STRING)
if (NOT CMAKE_BUILD_TYPE)
    set (CMAKE_BUILD_TYPE Release)
endif ()

set (CMAKE_CXX_LINK_FLAGS "-O3")
set (CMAKE_CXX_FLAGS_RELEASE "-O3")
set (CMAKE_CXX_FLAGS_DEBUG "-ggdb")
set (CMAKE_DEBUG_POSTFIX "-g")
add_definitions(-std=c++11)


set(BOOST_COMPONENTS thread system)

if (LIBDVID_INCLUDE)
    include_directories(AFTER ${LIBDVID_INCLUDE_DIR})
endif()

FIND_PACKAGE(Boost REQUIRED COMPONENTS ${BOOST_COMPONENTS})
include_directories(AFTER ${Boost_INCLUDE_DIR})
if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set (PLATFORM_DYLIB_EXTENSION "dylib")
else()
    set (PLATFORM_DYLIB_EXTENSION "so")
endif()
if (NOT Boost_LIBRARY_DIR_RELEASE)
    set(Boost_LIBRARY_DIR_RELEASE ${Boost_LIBRARY_DIR})
endif()
set (boost_LIBS ${Boost_LIBRARY_DIR_RELEASE}/libboost_thread.${PLATFORM_DYLIB_EXTENSION} ${Boost_LIBRARY_DIR_RELEASE}/libboost_system.${PLATFORM_DYLIB_EXTENSION})
if (LIBDVID_WRAP_PYTHON)
    set(boostpython_LIB ${Boost_LIBRARY_DIR_RELEASE}/libboost_python.${PLATFORM_DYLIB_EXTENSION})
endif()


FIND_PACKAGE(PNG REQUIRED)
FIND_PACKAGE(JPEG REQUIRED)
FIND_PACKAGE(CURL REQUIRED)
FIND_LIBRARY(LZ4_LIBRARY lz4)
FIND_LIBRARY(DVIDCPP_LIBRARY dvidcpp)

if (NOT LZ4_LIBRARY)
    message(FATAL_ERROR "*** Could not find lz4 library ***")
endif()

if (UNIX AND NOT APPLE)
    include (FindPkgConfig)
    PKG_SEARCH_MODULE(JSON REQUIRED jsoncpp)
    include_directories(BEFORE ${JSON_INCLUDE_DIRS})
else()
    FIND_LIBRARY(JSON_LIBRARIES jsoncpp)

    if (NOT JSON_LIBRARIES)
        message(FATAL_ERROR "*** Could not find jsoncpp library ***")
    endif()
endif()

#set (support_LIBS ${LIBDVIDCPP_LIBRARIES} ${JSON_LIBRARIES} ${boost_LIBS} ${PNG_LIBRARIES} ${CURL_LIBRARIES} ${JPEG_LIBRARIES} ${LZ4_LIBRARY})
set (support_LIBS ${DVIDCPP_LIBRARY})

include_directories (BEFORE ${CMAKE_SOURCE_DIR}/lowtis ${CMAKE_SOURCE_DIR})

if (NOT MSVC)
    # The -fPIC flag is necessary for "relocatable" code that might be included in an .so
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
endif()

# Compile lowtis library components
add_library (lowtis SHARED 
             src/BlockCache.cpp
             src/BlockFetchFactory.cpp
             src/DVIDBlockFetch.cpp
             src/lowtis.cpp)

target_link_libraries (lowtis ${support_LIBS})

# config file for CMake FIND_PACKAGE command
set (lowtis_version "0.1.0")
set(CONF_INCLUDE_DIRS "${PROJECT_SOURCE_DIR}")
export(TARGETS lowtis FILE "${PROJECT_BINARY_DIR}/lowtisTargets.cmake")

CONFIGURE_FILE(${PROJECT_SOURCE_DIR}/config/lowtisConfig.cmake.in
    ${PROJECT_BINARY_DIR}/lowtisConfig.cmake
    @ONLY IMMEDIATE)
CONFIGURE_FILE(${PROJECT_SOURCE_DIR}/config/lowtisConfigVersion.cmake.in
    ${PROJECT_BINARY_DIR}/lowtisConfigVersion.cmake
    @ONLY IMMEDIATE)

# installation for library
INSTALL(DIRECTORY ${PROJECT_SOURCE_DIR}/lowtis DESTINATION include)
INSTALL(TARGETS lowtis DESTINATION lib)
INSTALL(FILES ${PROJECT_BINARY_DIR}/lowtisConfig.cmake
${PROJECT_BINARY_DIR}/lowtisConfigVersion.cmake ${PROJECT_BINARY_DIR}/lowtisTargets.cmake
DESTINATION lib${LIB_SUFFIX}/lowtis)



