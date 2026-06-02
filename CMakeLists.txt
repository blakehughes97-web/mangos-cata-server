# MaNGOS is a full featured server for World of Warcraft, supporting
# the following clients: 1.12.x, 2.4.3, 3.3.5a, 4.3.4a and 5.4.8
#
# Copyright (C) 2005-2025 MaNGOS <https://www.getmangos.eu>
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 2 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA  02111-1307  USA

cmake_minimum_required(VERSION 3.18...3.27 FATAL_ERROR)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

cmake_policy(SET CMP0028 NEW)
cmake_policy(SET CMP0054 NEW)
cmake_policy(SET CMP0042 NEW)
cmake_policy(SET CMP0048 NEW)
cmake_policy(SET CMP0063 NEW)

set(CMAKE_CXX_STANDARD          17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS        OFF)
set(CMAKE_C_STANDARD          11)
set(CMAKE_C_STANDARD_REQUIRED ON)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

#==================================================================================
# Define available cmake options below
option(BUILD_MANGOSD        "Build the main server"                         ON)
option(BUILD_REALMD         "Build the login server"                        ON)
option(BUILD_TOOLS          "Build the map/vmap/mmap extractors"            ON)
option(USE_STORMLIB         "Use StormLib for reading MPQs"                 ON)
option(SCRIPT_LIB_ELUNA     "Compile with support for Eluna scripts"        ON)
option(SCRIPT_LIB_SD3       "Compile with support for ScriptDev3 scripts"   ON)
option(PLAYERBOTS           "Enable Player Bots"                            OFF)
option(SOAP                 "Enable remote access via SOAP"                 OFF)
option(PCH                  "Enable precompiled headers"                    ON)
option(DEBUG                "Enable debug build (only on non IDEs)"         OFF)
#==================================================================================

message("")
message(
"This script builds the MaNGOS server.
Options that can be used in order to configure the process:
General:
CMAKE_INSTALL_PREFIX    Path where the server should be installed to
BUILD_MANGOSD           Build the main server
BUILD_REALMD            Build the login server
BUILD_TOOLS             Build the map/vmap/mmap extractors
USE_STORMLIB            Use StormLib for reading MPQs
SOAP                    Enable remote access via SOAP
PCH                     Enable use of precompiled headers
DEBUG                   Debug build, only for systems without IDE (Linux, *BSD)
Scripting engines:
SCRIPT_LIB_ELUNA        Compile with support for Eluna scripts
SCRIPT_LIB_SD3          Compile with support for ScriptDev3 scripts
Modules:
PLAYERBOTS              Enable Player Bots

To set an option simply type -D<OPTION>=<VALUE> after 'cmake <srcs>'.
Also, you can specify the generator with -G. See 'cmake --help' for more details
For example: cmake .. -DCMAKE_INSTALL_PREFIX=/opt/mangos -DSCRIPT_LIB_SD3=0"
)
message("")

#==================================================================================#
project(MaNGOS VERSION 0.22.0)

if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX ${CMAKE_BINARY_DIR}/install CACHE PATH "Default install directory" FORCE)
endif()

if(DEBUG)
  set(CMAKE_BUILD_TYPE Debug)
else()
  set(CMAKE_BUILD_TYPE Release)
endif()

if(UNIX)
  set(BIN_DIR ${CMAKE_INSTALL_PREFIX}/bin)
  if (NOT CONF_INSTALL_DIR)
    set(CONF_INSTALL_DIR ../etc)
  endif()
else()
  set(BIN_DIR ${CMAKE_INSTALL_PREFIX})
  if (NOT CONF_INSTALL_DIR)
    set(CONF_INSTALL_DIR .)
  endif()
endif()

if(NOT WITHOUT_GIT)
  find_package(Git)
endif()

find_package(Threads REQUIRED)

#==================================================================================
# Force Local MariaDB Installation (C:\Mangos-Bots\mariadb-10.6.3-winx64)
#==================================================================================
set(MARIADB_ROOT "${CMAKE_SOURCE_DIR}/mariadb-10.6.3-winx64")

# Auto-detect include path
if(EXISTS "${MARIADB_ROOT}/include/mysql.h")
    set(MYSQL_INCLUDE_DIR "${MARIADB_ROOT}/include" CACHE PATH "MariaDB Include" FORCE)
else()
    set(MYSQL_INCLUDE_DIR "${MARIADB_ROOT}/include/mysql" CACHE PATH "MariaDB Include" FORCE)
endif()

# Auto-detect library file
if(EXISTS "${MARIADB_ROOT}/lib/libmariadb.lib")
    set(MYSQL_LIBRARY "${MARIADB_ROOT}/lib/libmariadb.lib" CACHE FILEPATH "MariaDB Library" FORCE)
elseif(EXISTS "${MARIADB_ROOT}/lib/libmysql.lib")
    set(MYSQL_LIBRARY "${MARIADB_ROOT}/lib/libmysql.lib" CACHE FILEPATH "MariaDB Library" FORCE)
elseif(EXISTS "${MARIADB_ROOT}/lib/mariadbclient.lib")
    set(MYSQL_LIBRARY "${MARIADB_ROOT}/lib/mariadbclient.lib" CACHE FILEPATH "MariaDB Library" FORCE)
endif()

set(MYSQL_FOUND TRUE CACHE BOOL "MySQL Found" FORCE)

find_package(MySQL   REQUIRED)
find_package(DL      REQUIRED)
find_package(ZLIB  QUIET)
find_package(BZip2 QUIET)
find_package(OpenSSL REQUIRED)

include(${CMAKE_SOURCE_DIR}/cmake/GenRevision.cmake)
include(${CMAKE_SOURCE_DIR}/cmake/EnsureVersion.cmake)
include(${CMAKE_SOURCE_DIR}/cmake/MangosParams.cmake)
include(${CMAKE_SOURCE_DIR}/cmake/SetDefinitions.cmake)

if(PCH)
  include(${CMAKE_SOURCE_DIR}/cmake/PCHSupport.cmake)
endif()

add_subdirectory(dep)
add_subdirectory(src)

#==================================================================================
# Windows Platform Dependencies (Fixes MariaDB/SSL/Zlib linker errors)
# DO NOT REMOVE - Required for successful linking on Windows
#==================================================================================
if(WIN32)
  # Link required Windows libraries to mangosd and realmd targets
  if(TARGET mangosd)
    target_link_libraries(mangosd PRIVATE
      shlwapi     # PathRemoveFileSpecA
      secur32     # EncryptMessage, DecryptMessage, InitializeSecurityContextA, etc.
      crypt32     # Cert*, Crypt* functions
      ws2_32      # Network socket fallbacks
      zlib        # compress/uncompress
    )
  endif()
  if(TARGET realmd)
    target_link_libraries(realmd PRIVATE
      shlwapi
      secur32
      crypt32
      ws2_32
      zlib
    )
  endif()

  #==================================================================================
  # Auto-copy MariaDB DLL to the install folder
  #==================================================================================
  set(MARIADB_DLL "")
  if(EXISTS "${MARIADB_ROOT}/lib/libmariadb.dll")
    set(MARIADB_DLL "${MARIADB_ROOT}/lib/libmariadb.dll")
  elseif(EXISTS "${MARIADB_ROOT}/lib/libmysql.dll")
    set(MARIADB_DLL "${MARIADB_ROOT}/lib/libmysql.dll")
  endif()

  if(MARIADB_DLL)
    install(FILES "${MARIADB_DLL}" DESTINATION "${BIN_DIR}" COMPONENT runtime)
    message(STATUS "MariaDB DLL will be auto-copied to install folder: ${MARIADB_DLL}")
  else()
    message(WARNING "MariaDB DLL not found in ${MARIADB_ROOT}/lib/. You may need to copy it manually.")
  endif()
endif()

include(${CMAKE_SOURCE_DIR}/cmake/StatusInfo.cmake)