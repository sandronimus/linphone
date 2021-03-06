############################################################################
# CMakeLists.txt
# Copyright (C) 2014  Belledonne Communications, Grenoble France
#
############################################################################
#
# This program is free software; you can redistribute it and/or
# modify it under the terms of the GNU General Public License
# as published by the Free Software Foundation; either version 2
# of the License, or (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 59 Temple Place - Suite 330, Boston, MA  02111-1307, USA.
#
############################################################################

cmake_minimum_required(VERSION 3.0)
project(LINPHONE C CXX)


set(LINPHONE_MAJOR_VERSION "3")
set(LINPHONE_MINOR_VERSION "9")
set(LINPHONE_MICRO_VERSION "1")
set(LINPHONE_VERSION "${LINPHONE_MAJOR_VERSION}.${LINPHONE_MINOR_VERSION}.${LINPHONE_MICRO_VERSION}")
set(LINPHONE_SO_VERSION "8")

file(GLOB LINPHONE_PO_FILES RELATIVE "${CMAKE_CURRENT_LIST_DIR}/po" "${CMAKE_CURRENT_LIST_DIR}/po/*.po")
string(REGEX REPLACE "([a-zA-Z_]+)\\.po" "\\1" LINPHONE_ALL_LANGS_LIST "${LINPHONE_PO_FILES}")
string(REPLACE ";" " " LINPHONE_ALL_LANGS "${LINPHONE_ALL_LANGS_LIST}")

include(CMakeDependentOption)

option(ENABLE_STATIC "Build static library (default is shared library)." NO)
option(ENABLE_CONSOLE_UI "Turn on or off compilation of console interface." YES)
option(ENABLE_DATE "Use build date in internal version number." NO)
option(ENABLE_DOC "Enable documentation generation with Doxygen." YES)
option(ENABLE_GTK_UI "Turn on or off compilation of gtk interface." YES)
option(ENABLE_LDAP "Enable LDAP support." NO)
option(ENABLE_LIME "Enable Instant Messaging Encryption." NO)
option(ENABLE_MSG_STORAGE "Turn on compilation of message storage." YES)
cmake_dependent_option(ENABLE_NOTIFY "Enable libnotify support." YES "ENABLE_GTK_UI;NOT APPLE" NO)
option(ENABLE_RELATIVE_PREFIX "Find resources relatively to the installation directory." NO)
option(ENABLE_STRICT "Build with strict compile options." YES)
option(ENABLE_TOOLS "Turn on or off compilation of tools." YES)
option(ENABLE_TUNNEL "Turn on compilation of tunnel support." NO)
option(ENABLE_TUTORIALS "Enable compilation of tutorials." YES)
option(ENABLE_UNIT_TESTS "Enable compilation of unit tests." YES)
option(ENABLE_UPNP "Build with uPnP support." YES)
option(ENABLE_VIDEO "Build with video support." YES)
cmake_dependent_option(ENABLE_ASSISTANT "Turn on assistant compiling." YES "ENABLE_GTK_UI" NO)
option(ENABLE_DEBUG_LOGS "Turn on or off debug level logs." NO)
option(ENABLE_NLS "Build with internationalisation support" YES)
option(ENABLE_CALL_LOGS_STORAGE "Turn on compilation of call logs storage." YES)


macro(apply_compile_flags SOURCE_FILES)
	if(${SOURCE_FILES})
		set(options "")
		foreach(a ${ARGV})
			if(STRICT_OPTIONS_${a})
				string(REPLACE ";" " " options_${a} "${STRICT_OPTIONS_${a}}")
				set(options "${options} ${options_${a}}")
			endif()
		endforeach()
		if(options)
			set_source_files_properties(${${SOURCE_FILES}} PROPERTIES COMPILE_FLAGS "${options}")
		endif()
	endif()
endmacro()


list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

include(CheckSymbolExists)
include(CMakePushCheckState)

set(MSVC_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/include/MSVC")
if(MSVC)
	list(APPEND CMAKE_REQUIRED_INCLUDES "${MSVC_INCLUDE_DIR}")
endif()

# find_package should be invoked here to check for libraries - however do NOT
# call include_directories here (see below)

if(LINPHONE_BUILDER_GROUP_EXTERNAL_SOURCE_PATH_BUILDERS)
	include("${EP_bellesip_CONFIG_DIR}/BelleSIPConfig.cmake")
	include("${EP_ms2_CONFIG_DIR}/Mediastreamer2Config.cmake")
else()
	find_package(BelleSIP REQUIRED)
	find_package(Mediastreamer2 REQUIRED)
endif()
find_package(XML2 REQUIRED)
find_package(Zlib)
if(ENABLE_UNIT_TESTS)
	find_package(CUnit)
	if(CUNIT_FOUND)
		cmake_push_check_state(RESET)
		list(APPEND CMAKE_REQUIRED_INCLUDES ${CUNIT_INCLUDE_DIRS})
		list(APPEND CMAKE_REQUIRED_LIBRARIES ${CUNIT_LIBRARIES})
		check_symbol_exists("CU_get_suite" "CUnit/CUnit.h" HAVE_CU_GET_SUITE)
		check_symbol_exists("CU_curses_run_tests" "CUnit/CUnit.h" HAVE_CU_CURSES)
		cmake_pop_check_state()
	else()
		message(WARNING "Could not find the cunit library!")
		set(ENABLE_UNIT_TESTS OFF CACHE BOOL "Enable compilation of unit tests." FORCE)
	endif()
endif()
if(ENABLE_TUNNEL)
	if(LINPHONE_BUILDER_GROUP_EXTERNAL_SOURCE_PATH_BUILDERS)
		include("${EP_tunnel_CONFIG_DIR}/TunnelConfig.cmake")
	else()
		find_package(Tunnel)
	endif()
	if(NOT TUNNEL_FOUND)
		message(WARNING "Could not find the tunnel library!")
		set(ENABLE_TUNNEL OFF CACHE BOOL "Enable tunnel support." FORCE)
	endif()
endif()
if(ENABLE_MSG_STORAGE)
	find_package(Sqlite3 REQUIRED)
endif()
if(ENABLE_NOTIFY)
	find_package(Notify)
	if(NOTIFY_FOUND)
		set(HAVE_NOTIFY4 1)
	else()
		message(WARNING "Could not find the notify library!")
		set(ENABLE_NOTIFY OFF CACHE BOOL "Enable libnotify support." FORCE)
	endif()
endif()
if(ENABLE_GTK_UI)
	if(WIN32)
		set(GTK2_ADDITIONAL_SUFFIXES "../lib/glib-2.0/include" "../lib/gtk-2.0/include")
	endif()
	find_package(GTK2 2.18 REQUIRED gtk)
	if(ENABLE_ASSISTANT AND GTK2_VERSION VERSION_LESS 2.22)
		message(WARNING "You need at least GTK 2.22 to enable the assistant")
		set(ENABLE_ASSISTANT OFF CACHE BOOL "Turn on assistant compiling." FORCE)
	endif()
	if(APPLE)
		find_package(GtkMacIntegration)
		if(GTKMACINTEGRATION_FOUND)
			set(HAVE_GTK_OSX 1)
			add_definitions("${GTKMACINTEGRATION_CPPFLAGS}")
		else()
			message(WARNING "gtk-mac-integration not found. Please install gtk-osx-application package.")
		endif()
	endif()
endif()
if(ENABLE_ASSISTANT)
	set(BUILD_WIZARD 1)
endif()
if(ENABLE_NLS)
	find_package(Gettext REQUIRED)
	find_package(Intl REQUIRED)
endif()
if(ENABLE_CALL_LOGS_STORAGE)
	find_package(Sqlite3 REQUIRED)
endif()
if(ENABLE_LIME)
	set(HAVE_LIME 1)
endif()

if(UNIX AND NOT APPLE)
	include(CheckIncludeFiles)
	check_include_files(libudev.h HAVE_LIBUDEV_H)
endif()

# include_directories must be called only UNDER THIS LINE in order to use our
# projects submodules first (we do NOT want to have system headers in first position)
include_directories(
	include/
	coreapi/
	${CMAKE_CURRENT_BINARY_DIR}
	${CMAKE_CURRENT_BINARY_DIR}/coreapi/
	${BELLESIP_INCLUDE_DIRS}
	${MEDIASTREAMER2_INCLUDE_DIRS}
)
if(ENABLE_TUNNEL)
	include_directories(${TUNNEL_INCLUDE_DIRS})
endif()

include_directories(${XML2_INCLUDE_DIRS})

if(ZLIB_FOUND)
	include_directories(${ZLIB_INCLUDE_DIRS})
	set(HAVE_ZLIB 1)
endif()
if(SQLITE3_FOUND)
	include_directories(${SQLITE3_INCLUDE_DIRS})
	if(ENABLE_MSG_STORAGE)
		add_definitions("-DMSG_STORAGE_ENABLED")
	endif()
	if(ENABLE_CALL_LOGS_STORAGE)
		add_definitions("-DCALL_LOGS_STORAGE_ENABLED")
	endif()
endif()
if(INTL_FOUND)
	set(HAVE_INTL 1)
	include_directories(${INTL_INCLUDE_DIRS})
endif()
if(MSVC)
	include_directories(${MSVC_INCLUDE_DIR})
endif()

add_definitions("-DIN_LINPHONE")
if(ENABLE_DEBUG_LOGS)
	add_definitions("-DDEBUG")
endif()

set(STRICT_OPTIONS_CPP )
set(STRICT_OPTIONS_C )
set(STRICT_OPTIONS_CXX "-std=c++11")
set(STRICT_OPTIONS_OBJC )
if(NOT MSVC)
	list(APPEND STRICT_OPTIONS_CPP "-Wall" "-Wuninitialized" "-Wno-error=deprecated-declarations")
	list(APPEND STRICT_OPTIONS_C "-Wdeclaration-after-statement" "-Wstrict-prototypes" "-Wno-error=strict-prototypes")
	if(CMAKE_C_COMPILER_ID STREQUAL "Clang")
		list(APPEND STRICT_OPTIONS_CPP "-Qunused-arguments" "-Wno-array-bounds")
	endif()
	if(APPLE)
		list(APPEND STRICT_OPTIONS_CPP "-Wno-error=unknown-warning-option" "-Qunused-arguments" "-Wno-tautological-compare" "-Wno-unused-function" "-Wno-array-bounds")
	endif()
	if(ENABLE_STRICT)
		list(APPEND STRICT_OPTIONS_CPP "-Werror" "-fno-strict-aliasing")
	endif()
endif()
if(CMAKE_SYSTEM_NAME STREQUAL "WindowsStore")
	list(APPEND STRICT_OPTIONS_CPP "/wd4996")
endif()
if(STRICT_OPTIONS_CPP)
	list(REMOVE_DUPLICATES STRICT_OPTIONS_CPP)
endif()
if(STRICT_OPTIONS_C)
	list(REMOVE_DUPLICATES STRICT_OPTIONS_C)
endif()


set(GETTEXT_PACKAGE "linphone")
if(ENABLE_RELATIVE_PREFIX)
	set(LINPHONE_DATA_DIR ".")
else()
	set(LINPHONE_DATA_DIR "${CMAKE_INSTALL_PREFIX}")
endif()
set(LINPHONE_PLUGINS_DIR "${LINPHONE_DATA_DIR}/lib/liblinphone/plugins")
if(WIN32)
	set(LINPHONE_CONFIG_DIR "Linphone")
endif()
set(PACKAGE_LOCALE_DIR "${LINPHONE_DATA_DIR}/share/locale")
set(PACKAGE_DATA_DIR "${LINPHONE_DATA_DIR}/share")
set(PACKAGE_SOUND_DIR "${LINPHONE_DATA_DIR}/share/sounds/linphone")
set(PACKAGE_RING_DIR "${PACKAGE_SOUND_DIR}/rings")
set(PACKAGE_FREEDESKTOP_DIR "${PACKAGE_DATA_DIR}/applications")
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/config.h)
set_source_files_properties(${CMAKE_CURRENT_BINARY_DIR}/config.h PROPERTIES GENERATED ON)
add_definitions(-DHAVE_CONFIG_H)


if(ENABLE_VIDEO)
	add_definitions(-DVIDEO_ENABLED)
endif()


if(LINPHONE_BUILDER_GROUP_EXTERNAL_SOURCE_PATH_BUILDERS)
	set(EXPORT_TARGETS_NAME "LinphoneBuilder")
else()
	set(EXPORT_TARGETS_NAME "Linphone")
endif()


add_subdirectory(coreapi)
add_subdirectory(share)
if(ENABLE_CONSOLE_UI)
	add_subdirectory(console)
endif()
if(ENABLE_GTK_UI)
	add_subdirectory(gtk)
	add_subdirectory(pixmaps)
	add_subdirectory(po)
endif()
if(ENABLE_TOOLS)
	add_subdirectory(tools)
endif()
if(ENABLE_UNIT_TESTS)
	add_subdirectory(tester)
endif()


include(CMakePackageConfigHelpers)
write_basic_package_version_file(
	"${CMAKE_CURRENT_BINARY_DIR}/LinphoneConfigVersion.cmake"
	VERSION ${LINPHONE_VERSION}
	COMPATIBILITY AnyNewerVersion
)
export(EXPORT ${EXPORT_TARGETS_NAME}Targets
	FILE "${CMAKE_CURRENT_BINARY_DIR}/LinphoneTargets.cmake"
)
configure_file(cmake/LinphoneConfig.cmake.in
	"${CMAKE_CURRENT_BINARY_DIR}/LinphoneConfig.cmake"
	@ONLY
)

set(ConfigPackageLocation lib/cmake/Linphone)
install(EXPORT ${EXPORT_TARGETS_NAME}Targets
	FILE LinphoneTargets.cmake
	DESTINATION ${ConfigPackageLocation}
)
install(FILES
	"${CMAKE_CURRENT_BINARY_DIR}/LinphoneConfig.cmake"
	"${CMAKE_CURRENT_BINARY_DIR}/LinphoneConfigVersion.cmake"
	DESTINATION ${ConfigPackageLocation}
)
