####################################
# General project definition
####################################

CMAKE_MINIMUM_REQUIRED(VERSION 3.5.2 FATAL_ERROR)

### CMP0025   Compiler id for Apple Clang is now AppleClang.
### CMP0042   MACOSX_RPATH is enabled by default.

FOREACH (p
        CMP0025 # CMake 3.0
        CMP0042 # CMake 3.0
        )
    IF (POLICY ${p})
        cmake_policy(SET ${p} NEW)
    ENDIF ()
endforeach ()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake) # project specific cmake dir
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/../cmake)

project(InfoLogger)

# Definition of modules to be built
# by default, build all
# if some specific modules are specified, build only these ones
set(INFOLOGGER_BUILD_ALL 1)
if(INFOLOGGER_BUILD_LIB) 
  set(INFOLOGGER_BUILD_ALL 0)
  message("Enabling libInfologger.") 
endif()
if(INFOLOGGER_BUILD_ALL)
  message("Enabling all InfoLogger modules.")
  set(INFOLOGGER_BUILD_LIB 0)
endif()



# Load some basic macros which are needed later on
include(O2Utils)
include(InfoLoggerDependencies)

# Set the default build type to "RelWithDebInfo"
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "RelWithDebInfo"
            CACHE
            STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel Coverage."
            FORCE)
endif(NOT CMAKE_BUILD_TYPE)

# Set the version number of your project here (format is MAJOR.MINOR.PATCHLEVEL - e.g. 1.0.0)
set(VERSION_MAJOR "1")
set(VERSION_MINOR "0")
set(VERSION_PATCH "5")
set(VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH})

# C++14
IF (CMAKE_VERSION VERSION_LESS 3.1)
    include(CheckCXXCompilerFlag)
    CHECK_CXX_COMPILER_FLAG(-std=c++14 COMPILER_SUPPORTS_CXX14)
    if (COMPILER_SUPPORTS_CXX14)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14")
    else ()
        message(ERROR "The compiler ${CMAKE_CXX_COMPILER} has no C++14 support. Please use a different C++ compiler.")
    endif ()
ELSE ()
    set(CMAKE_CXX_STANDARD 14)  # proper way in CMake >= 3.1
ENDIF ()

# Add compiler flags for warnings and (more importantly) fPIC and debug symbols
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -pedantic -Wextra -fPIC")


if(INFOLOGGER_BUILD_ALL)

####################################
# Module, library and executable definition
####################################

set(MODULE_NAME "InfoLogger")
O2_SETUP(NAME ${MODULE_NAME})

set_source_files_properties(src/simplelog.c PROPERTIES LANGUAGE CXX )

set(SRCS
  src/InfoLogger.cxx
  src/infoLoggerMessage.c
#  InfoLogger.c
  src/InfoLoggerClient.cxx
#  SimpleLog.cxx
  src/InfoLoggerMessageHelper.cxx
  src/utility.c
  src/transport_server.c
  src/transport_client.c
  src/transport_files.c
  src/simplelog.c
  src/transport_proxy.c
  src/permanentFIFO.c
  src/infoLoggerMessageDecode.c
  src/InfoLoggerDispatch.cxx
  src/InfoLoggerDispatchBrowser.cxx
  src/InfoLoggerMessageList.cxx
  src/ConfigInfoLoggerServer.cxx
  src/InfoLoggerContext.cxx
  )


if(MYSQL_FOUND)
    list(APPEND SRCS src/InfoLoggerDispatchSQL.cxx)
endif()


# this one is included in other files
list(REMOVE_ITEM SRCS src/infoLoggerMessage.c)

# Produce the final Version.h using template Version.h.in and substituting variables.
# We don't want to polute our source tree with it, thus putting it in binary tree.
configure_file("include/${MODULE_NAME}/Version.h.in"
        "${CMAKE_CURRENT_BINARY_DIR}/include/${MODULE_NAME}/Version.h"
        @ONLY
        )

include_directories(
        ${CMAKE_CURRENT_BINARY_DIR}/include include ${Boost_INCLUDE_DIRS} ${Common_INCLUDE_DIRS}
)

set(HEADERS
  ${CMAKE_CURRENT_BINARY_DIR}/include/${MODULE_NAME}/Version.h
  include/${MODULE_NAME}/InfoLogger.hxx
  include/${MODULE_NAME}/InfoLoggerFMQ.hxx
  include/${MODULE_NAME}/InfoLogger.h
)

set(LIBRARY_NAME ${MODULE_NAME})

if(MYSQL_FOUND)
    set(BUCKET_NAME o2_infologger_bucket_with_mysql)
else()
    set(BUCKET_NAME o2_infologger_bucket)
endif()

O2_GENERATE_LIBRARY()

add_subdirectory(doc)

O2_GENERATE_EXECUTABLE(
        EXE_NAME infoLoggerServer
        SOURCES src/infoLoggerServer.cxx
        MODULE_LIBRARY_NAME ${LIBRARY_NAME}
        BUCKET_NAME ${BUCKET_NAME}
)

O2_GENERATE_EXECUTABLE(
        EXE_NAME infoLoggerD
        SOURCES src/infoLoggerD.cxx
        MODULE_LIBRARY_NAME ${LIBRARY_NAME}
        BUCKET_NAME ${BUCKET_NAME}
)

O2_GENERATE_EXECUTABLE(
        EXE_NAME log
        SOURCES src/log.cxx
        MODULE_LIBRARY_NAME ${LIBRARY_NAME}
        BUCKET_NAME ${BUCKET_NAME}
)

if(MYSQL_FOUND)
   O2_GENERATE_EXECUTABLE(
           EXE_NAME infoLoggerAdminDB
           SOURCES src/infoLoggerAdminDB.cxx
           MODULE_LIBRARY_NAME ${LIBRARY_NAME}
           BUCKET_NAME ${BUCKET_NAME}
   )
endif()



enable_testing()

set(TEST_SRCS
  test/testInfoLogger.cxx
  test/testInfoLoggerPerf.cxx
)

O2_GENERATE_TESTS(
 MODULE_LIBRARY_NAME ${LIBRARY_NAME}
 BUCKET_NAME ${BUCKET_NAME}
 TEST_SRCS ${TEST_SRCS}
)


link_directories(
     ${LIBRARY_OUTPUT_PATH}
)



# Some extra targets to build standalone executables not relying on other o2 shared libs in bin-standalone
# This is added optionally with cmake -DBUILD_STANDALONE=1 ...
if(BUILD_STANDALONE)
  include(ExternalProject)
  ExternalProject_Add (Common
     GIT_REPOSITORY "https://github.com/AliceO2Group/Common.git"
     GIT_TAG "master"
     LOG_DOWNLOAD 1
     UPDATE_COMMAND ""
     PATCH_COMMAND ""
     BUILD_COMMAND ""
     CONFIGURE_COMMAND ""
     TEST_COMMAND ""
     INSTALL_COMMAND ""
  )
  add_dependencies(InfoLogger Common)

  ExternalProject_Get_Property(Common source_dir)
  include_directories(${source_dir}/include)

  foreach (modul Configuration Daemon LineBuffer SimpleLog Thread Timer)
      add_library(
        objCommon${modul} OBJECT
        ${source_dir}/src/${modul}.cxx
      )
      set_source_files_properties(${source_dir}/src/${modul}.cxx PROPERTIES GENERATED TRUE)
      set_target_properties(objCommon${modul} PROPERTIES
        POSITION_INDEPENDENT_CODE ON        
      )
      add_dependencies(objCommon${modul} Common)      
  endforeach (modul)
  

  add_library (objInfoLoggerClient OBJECT
      src/InfoLogger.cxx
      src/InfoLoggerContext.cxx
      src/InfoLoggerClient.cxx
      src/infoLoggerMessageDecode.c
      src/InfoLoggerMessageHelper.cxx
      src/utility.c
      src/simplelog.c
  )
  set_target_properties(objInfoLoggerClient PROPERTIES
    POSITION_INDEPENDENT_CODE ON
  )
  
  set (INFOLOGGER_STANDALONE_OBJ
      $<TARGET_OBJECTS:objCommonConfiguration>
      $<TARGET_OBJECTS:objCommonSimpleLog>
      $<TARGET_OBJECTS:objInfoLoggerClient>
  )
  add_library (InfoLogger-standalone SHARED
    ${INFOLOGGER_STANDALONE_OBJ}
  )
  set_target_properties(InfoLogger-standalone PROPERTIES
      POSITION_INDEPENDENT_CODE ON
  )
  
  add_library (InfoLogger-standalone-static STATIC
    ${INFOLOGGER_STANDALONE_OBJ}
  )
  set_target_properties( InfoLogger-standalone-static PROPERTIES
      OUTPUT_NAME InfoLogger-standalone
      POSITION_INDEPENDENT_CODE ON
  )
  
  add_library (objInfoLoggerTransport OBJECT
          src/permanentFIFO.c
          src/simplelog.c
          src/transport_client.c
          src/transport_files.c
          src/transport_proxy.c
          src/transport_server.c
          src/utility.c
  )
  set_target_properties( objInfoLoggerTransport PROPERTIES
      POSITION_INDEPENDENT_CODE ON
  )

  add_executable(
          infoLoggerD-s
          src/infoLoggerD.cxx
          $<TARGET_OBJECTS:objInfoLoggerTransport>
          $<TARGET_OBJECTS:objCommonConfiguration>
          $<TARGET_OBJECTS:objCommonSimpleLog>
          $<TARGET_OBJECTS:objCommonDaemon>
  )
  target_link_libraries(
          infoLoggerD-s
          pthread
  )
  
  # Library for Python
  if (PYTHONLIBS_FOUND)
    add_library (infoLoggerForPython SHARED
      src/infoLoggerForPython_wrap.cxx
      ${INFOLOGGER_STANDALONE_OBJ}
    )
    target_include_directories(
      infoLoggerForPython PUBLIC
      ${PYTHON_INCLUDE_PATH}
    )
    set_target_properties(infoLoggerForPython PROPERTIES PREFIX "_")
  endif ()

  # Library for Tcl
  if (TCL_FOUND)
    add_library (infoLoggerForTcl SHARED
      src/infoLoggerForTcl_wrap.cxx
      ${INFOLOGGER_STANDALONE_OBJ}
    )
    target_include_directories(
      infoLoggerForTcl PUBLIC
      ${TCL_INCLUDE_PATH}
    )
  endif ()
 
  add_executable(
          infoLoggerServer-s
          src/infoLoggerServer.cxx
          $<TARGET_OBJECTS:objInfoLoggerTransport>
          $<TARGET_OBJECTS:objCommonConfiguration>
          $<TARGET_OBJECTS:objCommonSimpleLog>
          $<TARGET_OBJECTS:objCommonDaemon>
          $<TARGET_OBJECTS:objCommonThread>
          src/InfoLoggerDispatch.cxx
          src/ConfigInfoLoggerServer.cxx
          src/InfoLoggerDispatchSQL.cxx
          src/InfoLoggerDispatchBrowser.cxx
          src/infoLoggerMessageDecode.c
          src/InfoLoggerMessageHelper.cxx
          src/InfoLoggerMessageList.cxx
  )
  target_include_directories(
          infoLoggerServer-s PUBLIC
          ${MYSQL_INCLUDE_DIRS}
  )
  target_link_libraries(
          infoLoggerServer-s
          pthread
          ${MYSQL_LIBRARIES}
  )
  add_executable(
          infoLoggerAdminDB-s
          $<TARGET_OBJECTS:objCommonConfiguration>
          $<TARGET_OBJECTS:objCommonSimpleLog>
          src/infoLoggerAdminDB.cxx
  )
  target_include_directories(
          infoLoggerAdminDB-s PUBLIC
          ${MYSQL_INCLUDE_DIRS}
  )
  target_link_libraries(
          infoLoggerAdminDB-s
          ${MYSQL_LIBRARIES}
  )
  add_executable(
          log-s
          src/log.cxx
          $<TARGET_OBJECTS:objCommonLineBuffer>
  )
  target_link_libraries(
          log-s
          InfoLogger-standalone-static
  )
  set_target_properties( infoLoggerD-s infoLoggerServer-s infoLoggerAdminDB-s log-s
      PROPERTIES
      RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin-standalone"
  )
  set_target_properties( infoLoggerD-s PROPERTIES
      OUTPUT_NAME infoLoggerD
  )
  set_target_properties( infoLoggerServer-s PROPERTIES
      OUTPUT_NAME infoLoggerServer
  )
  set_target_properties( log-s PROPERTIES
      OUTPUT_NAME log
  )
  set_target_properties( infoLoggerAdminDB-s PROPERTIES
      OUTPUT_NAME infoLoggerAdminDB
  )
endif(BUILD_STANDALONE)


# Install some extra files
install(
  FILES src/infoBrowser.tcl
  DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
  RENAME infoBrowser
  PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)

install(
  FILES newMysql.sh
  DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
  PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)


endif()


if(INFOLOGGER_BUILD_LIB)

  message("Generating makefile for libInfologger") 
  set(MODULE_NAME "InfoLogger")
  O2_SETUP(NAME ${MODULE_NAME})

  set_source_files_properties(src/simplelog.c PROPERTIES LANGUAGE CXX )

  set(SRCS
      src/InfoLogger.cxx
      src/InfoLoggerContext.cxx
      src/InfoLoggerClient.cxx
      src/infoLoggerMessageDecode.c
      src/InfoLoggerMessageHelper.cxx
      src/utility.c
      src/simplelog.c
  )

  # Produce the final Version.h using template Version.h.in and substituting variables.
  # We don't want to polute our source tree with it, thus putting it in binary tree.
  configure_file("include/${MODULE_NAME}/Version.h.in"
          "${CMAKE_CURRENT_BINARY_DIR}/include/${MODULE_NAME}/Version.h"
          @ONLY
          )

  set(HEADERS
    ${CMAKE_CURRENT_BINARY_DIR}/include/${MODULE_NAME}/Version.h
    include/${MODULE_NAME}/InfoLogger.hxx
    include/${MODULE_NAME}/InfoLoggerFMQ.hxx
  )
  
  include_directories(
          ${CMAKE_CURRENT_BINARY_DIR}/include include ${Boost_INCLUDE_DIRS} ${Common_INCLUDE_DIRS}
  )

  set(LIBRARY_NAME ${MODULE_NAME})
  set(BUCKET_NAME o2_libinfologger_bucket)

  O2_GENERATE_LIBRARY()  

  add_subdirectory(doc)

  set(BUCKET_NAME o2_infologger_bucket)

  enable_testing()

  set(TEST_SRCS
    test/testInfoLogger.cxx
    test/testInfoLoggerPerf.cxx
  )

  O2_GENERATE_TESTS(
   MODULE_LIBRARY_NAME ${LIBRARY_NAME}
   BUCKET_NAME ${BUCKET_NAME}
   TEST_SRCS ${TEST_SRCS}
  )


  link_directories(
       ${LIBRARY_OUTPUT_PATH}
  )
  
  message("Completed - Generating makefile for libInfologger")
endif()
