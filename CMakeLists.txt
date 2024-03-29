project(compton)
cmake_minimum_required(VERSION 2.8)

set(CPACK_PACKAGE_VERSION_MAJOR "0")
set(CPACK_PACKAGE_VERSION_MINOR "0")
set(CPACK_PACKAGE_VERSION_PATCH "0")

set(compton_SRCS src/compton.c) 
add_executable(compton ${compton_SRCS})

set(CMAKE_C_FLAGS_DEBUG "-ggdb")
set(CMAKE_C_FLAGS_RELEASE "-O2 -march=native")
set(CMAKE_C_FLAGS_RELWITHDEBINFO "-O2 -march=native -ggdb")

add_definitions("-Wall" "-std=c99")

# == Options ==

option(CONFIG_REGEX_PCRE "Enable PCRE regular expression support for blacklist entries (requires libpcre)" ON)
if (CONFIG_REGEX_PCRE)
	add_definitions("-DCONFIG_REGEX_PCRE")
endif ()

option(CONFIG_REGEX_PCRE_JIT "Use JIT support of libpcre)" ON)
if (CONFIG_REGEX_PCRE_JIT)
	add_definitions("-DCONFIG_REGEX_PCRE_JIT")
endif ()

option(CONFIG_LIBCONFIG "Enable configuration file parsing using libconfig" ON)
if (CONFIG_LIBCONFIG)
	add_definitions("-DCONFIG_LIBCONFIG")
endif ()

option(CONFIG_VSYNC_DRM "Enable DRM VSync support" ON)
if (CONFIG_VSYNC_DRM)
	add_definitions("-DCONFIG_VSYNC_DRM")
endif ()

option(CONFIG_VSYNC_OPENGL "Enable OpenGL VSync support" ON)
if (CONFIG_VSYNC_OPENGL)
	add_definitions("-DCONFIG_VSYNC_OPENGL")
endif ()

# == Find libraries ==

target_link_libraries(compton "-lm" "-lrt")

if (CONFIG_VSYNC_OPENGL)
  target_link_libraries(compton "-lGL")
endif ()

include(FindPkgConfig)

# --- Find X11 libs ---
set(X11_FIND_REQUIRED 1)
include(FindX11)

macro(X11LIB_CHK lib)
	if (NOT X11_${lib}_FOUND)
		message(FATAL_ERROR "Could not find lib${lib}.")
	endif ()
	target_link_libraries(compton "${X11_${lib}_LIB}")
endmacro ()

X11LIB_CHK(Xcomposite)
X11LIB_CHK(Xdamage)
X11LIB_CHK(Xext)
X11LIB_CHK(Xfixes)
X11LIB_CHK(Xrender)
X11LIB_CHK(Xrandr)

# --- Find libpcre ---
if (CONFIG_REGEX_PCRE)
	pkg_check_modules(LIBPCRE REQUIRED libpcre>=8.12)
	target_link_libraries(compton "${LIBPCRE_LDFLAGS}")
endif ()

# --- Find libconfig ---
if (CONFIG_LIBCONFIG)
	pkg_check_modules(LIBCONFIG REQUIRED libconfig>=1.3.2)
	target_link_libraries(compton "${LIBCONFIG_LDFLAGS}")
	if (LIBCONFIG_VERSION VERSION_LESS 1.4)
		add_definitions("-DCONFIG_LIBCONFIG_LEGACY")
		message(STATUS "libconfig-1.3* detected. Enable legacy mode.")
	endif ()
endif ()

# == Install ==
include(GNUInstallDirs)

install(TARGETS compton
	DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT prog)
install(FILES "${PROJECT_SOURCE_DIR}/bin/compton-trans"
	DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT prog)
install(FILES
	"${PROJECT_SOURCE_DIR}/man/compton.1"
	"${PROJECT_SOURCE_DIR}/man/compton-trans.1"
	DESTINATION "${CMAKE_INSTALL_MANDIR}" COMPONENT doc)
install(FILES
	"${PROJECT_SOURCE_DIR}/README.md"
	"${PROJECT_SOURCE_DIR}/LICENSE"
	DESTINATION "${CMAKE_INSTALL_DOCDIR}" COMPONENT doc)

# == CPack ==

if (NOT CPACK_SYSTEM_NAME)
	set(CPACK_SYSTEM_NAME "${CMAKE_SYSTEM_PROCESSOR}")
endif ()

set(CPACK_PACKAGE_DESCRIPTION "A lightweight X compositing window manager, fork of xcompmgr-dana.")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "A lightweight X compositing window manager")
set(CPACK_PACKAGE_CONTACT "nobody <devnull@example.com>")
set(CPACK_PACKAGE_DIRECTORY "${CMAKE_SOURCE_DIR}/release")
set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_SOURCE_DIR}/desc.txt")

# --- Package config ---
set(CPACK_GENERATOR "TBZ2" "DEB" "RPM")
set(CPACK_MONOLITHIC_INSTALL 1)
set(CPACK_STRIP_FILES 1)
set(CPACK_RESOURCE_FILE_LICENSE "LICENSE")
set(CPACK_RESOURCE_FILE_README "README.md")

# --- Source package config ---
set(CPACK_SOURCE_GENERATOR "TBZ2" "DEB" "RPM")
set(CPACK_SOURCE_IGNORE_FILES
	"/\\\\."
	"\\\\.bak$"
	"\\\\.o$"
	"\\\\~$"
	"\\\\.plist$"
	"/compton$"
	# Generated package files
	"\\\\.tar\\\\.gz$"
	"\\\\.tar\\\\.bz2$"
	"\\\\.deb$"
	"\\\\.rpm$"
	"compton-.*\\\\.sh$"
	# CMake files
	"/Makefile$"
	"/CMakeFiles/"
	"\\\\.cmake$"
	"CMakeCache\\\\.txt$"
	"/build/"
	"/release/"
	"\\\\.diff$"
	"/oprofile_data/"
	"/install_manifest\\\\.txt$"
)

# --- DEB package config ---
set(CPACK_DEBIAN_PACKAGE_SECTION "x11")
# The dependencies are unreliable, just an example here
set(CPACK_DEBIAN_PACKAGE_DEPENDS "libx11-6, libxext6, libxcomposite1, libxrender1, libxdamage1, libxfixes3, libpcre3, libconfig8")

# --- RPM package config ---
set(CPACK_RPM_PACKAGE_LICENSE "unknown")
# The dependencies are unreliable, just an example here
set(CPACK_RPM_PACKAGE_REQUIRES "libx11, libxext, libxcomposite, libxrender, libxdamage, libxfixes, libpcre, libconfig")

include(CPack)
