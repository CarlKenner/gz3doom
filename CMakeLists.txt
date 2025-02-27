cmake_minimum_required( VERSION 2.4 )
project(GZ3Doom)

list( APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR} )
include( CreateLaunchers )
include( FindPackageHandleStandardArgs )

# Generator expression are available some time in CMake 2.8. Due to
# cmake_minimum_required, we can assume a minor version of > 7 implies major >= 2
if(${CMAKE_MAJOR_VERSION} GREATER 2 OR ${CMAKE_MINOR_VERSION} GREATER 7)
	option( NO_GENERATOR_EXPRESSIONS "Disable generator expressions (for building pk3s with IDEs)." OFF )
else(${CMAKE_MAJOR_VERSION} GREATER 2 OR ${CMAKE_MINOR_VERSION} GREATER 7)
	set( NO_GENERATOR_EXPRESSIONS ON )
endif(${CMAKE_MAJOR_VERSION} GREATER 2 OR ${CMAKE_MINOR_VERSION} GREATER 7)

# Support cross compiling
option( FORCE_CROSSCOMPILE "Turn on cross compiling." NO )
if( FORCE_CROSSCOMPILE )
	set( CMAKE_CROSSCOMPILING TRUE )
endif( FORCE_CROSSCOMPILE )

if(CMAKE_CROSSCOMPILING)
	set(IMPORT_EXECUTABLES "IMPORTFILE-NOTFOUND" CACHE FILEPATH "Export file from native build.")
	include(${IMPORT_EXECUTABLES})
endif(CMAKE_CROSSCOMPILING)

# Simplify pk3 building, add_pk3(filename srcdirectory)
function( add_pk3 PK3_NAME PK3_DIR )
	get_target_property(ZIPDIR_EXE zipdir LOCATION)

	# Generate target name. Just use "pk3" for main pk3 target.
	string( REPLACE "." "_" PK3_TARGET ${PK3_NAME} )
	if( ${PK3_TARGET} STREQUAL "zdoom_pk3" )
		set( PK3_TARGET "pk3" )
	endif( ${PK3_TARGET} STREQUAL "zdoom_pk3" )

	if( NOT NO_GENERATOR_EXPRESSIONS AND NOT ZDOOM_OUTPUT_OLDSTYLE )
		add_custom_command( OUTPUT ${ZDOOM_OUTPUT_DIR}/${PK3_NAME}
			COMMAND ${ZIPDIR_EXE} -udf ${ZDOOM_OUTPUT_DIR}/${PK3_NAME} ${PK3_DIR}
			COMMAND ${CMAKE_COMMAND} -E copy_if_different ${ZDOOM_OUTPUT_DIR}/${PK3_NAME} $<TARGET_FILE_DIR:zdoom>/${PK3_NAME}
			DEPENDS zipdir )
	else( NOT NO_GENERATOR_EXPRESSIONS AND NOT ZDOOM_OUTPUT_OLDSTYLE )
		add_custom_command( OUTPUT ${ZDOOM_OUTPUT_DIR}/${PK3_NAME}
			COMMAND ${ZIPDIR_EXE} -udf ${ZDOOM_OUTPUT_DIR}/${PK3_NAME} ${PK3_DIR}
			DEPENDS zipdir )
	endif( NOT NO_GENERATOR_EXPRESSIONS AND NOT ZDOOM_OUTPUT_OLDSTYLE )

	# Touch the zipdir executable here so that the pk3s are forced to rebuild
	# each time since their dependecy has "changed."
	add_custom_target( ${PK3_TARGET} ALL
		COMMAND ${CMAKE_COMMAND} -E touch ${ZIPDIR_EXE}
		DEPENDS ${ZDOOM_OUTPUT_DIR}/${PK3_NAME} )

	install(FILES ${ZDOOM_OUTPUT_DIR}/${PK3_NAME} DESTINATION ".")
endfunction( add_pk3 )

# Macro for building libraries without debugging information
macro( make_release_only )
	set( CMAKE_C_FLAGS_MINSIZEREL ${CMAKE_C_FLAGS_RELEASE} )
	set( CMAKE_C_FLAGS_RELWITHDEBINFO ${CMAKE_C_FLAGS_RELEASE} )
	string( REPLACE "/MT " "/MTd " CMAKE_C_FLAGS_DEBUG ${CMAKE_C_FLAGS_RELEASE} )
	set( CMAKE_CXX_FLAGS_MINSIZEREL ${CMAKE_CXX_FLAGS_RELEASE} )
	set( CMAKE_CXX_FLAGS_RELWITHDEBINFO ${CMAKE_CXX_FLAGS_RELEASE} )
	string( REPLACE "/MT " "/MTd " CMAKE_CXX_FLAGS_DEBUG ${CMAKE_CXX_FLAGS_RELEASE} )
endmacro( make_release_only )

IF( NOT CMAKE_BUILD_TYPE )
	SET( CMAKE_BUILD_TYPE Debug CACHE STRING
		"Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel."
		FORCE )
ENDIF( NOT CMAKE_BUILD_TYPE )

set( ZDOOM_OUTPUT_DIR ${CMAKE_BINARY_DIR} CACHE PATH "Directory where zdoom.pk3 and the executable will be created." )
set( ZDOOM_EXE_NAME "GZ3Doom" CACHE FILEPATH "Name of the executable to create" )
if( MSVC )
	# Allow the user to use ZDOOM_OUTPUT_DIR as a single release point.
	# Use zdoom, zdoomd, zdoom64, and zdoomd64 for the binary names
	option( ZDOOM_OUTPUT_OLDSTYLE "Don't use Release/Debug directories." OFF )
else( MSVC )
	set( ZDOOM_OUTPUT_OLDSTYLE OFF )
endif( MSVC )

# Replacement variables for a possible long list of C/C++ compilers compatible with GCC
if( "${CMAKE_C_COMPILER_ID}" STREQUAL "GNU" OR "${CMAKE_C_COMPILER_ID}" STREQUAL "Clang" )
	set( ZD_CMAKE_COMPILER_IS_GNUC_COMPATIBLE TRUE )
else( "${CMAKE_C_COMPILER_ID}" STREQUAL "GNU" OR "${CMAKE_C_COMPILER_ID}" STREQUAL "Clang" )
	set( ZD_CMAKE_COMPILER_IS_GNUC_COMPATIBLE FALSE )
endif( "${CMAKE_C_COMPILER_ID}" STREQUAL "GNU" OR "${CMAKE_C_COMPILER_ID}" STREQUAL "Clang" )

if( "${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang" )
	set( ZD_CMAKE_COMPILER_IS_GNUCXX_COMPATIBLE TRUE )
else( "${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang" )
	set( ZD_CMAKE_COMPILER_IS_GNUCXX_COMPATIBLE FALSE )
endif( "${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang" )

if( ZD_CMAKE_COMPILER_IS_GNUCXX_COMPATIBLE )
	set( PROFILE 0 CACHE BOOL "Enable profiling with gprof for Debug and RelWithDebInfo build types." )
endif( ZD_CMAKE_COMPILER_IS_GNUCXX_COMPATIBLE )

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}")

find_package( BZip2 )
find_package( JPEG )
find_package( ZLIB )
# GME
find_path( GME_INCLUDE_DIR gme/gme.h )
find_library( GME_LIBRARIES gme )
mark_as_advanced( GME_INCLUDE_DIR GME_LIBRARIES )
FIND_PACKAGE_HANDLE_STANDARD_ARGS( GME
	REQUIRED_VARS GME_LIBRARIES GME_INCLUDE_DIR
)

if( MSVC )
	# Eliminate unreferenced functions and data
	# Perform identical COMDAT folding
	set( REL_LINKER_FLAGS "/opt:ref /opt:icf /nodefaultlib:msvcrt /TSAWARE" )

	# String pooling
	# Function-level linking
	# Disable run-time type information
	# Set floating point model to fast or the GL render will suffer for it.
	set( ALL_C_FLAGS "/GF /Gy /GR- /fp:fast" )
	
	if( CMAKE_SIZEOF_VOID_P MATCHES "4")
		# SSE2 option (mostly to switch it off in VC2012 and later where it's the default
		option (ZDOOM_USE_SSE2	"Use SSE2 instruction set")
		if (ZDOOM_USE_SSE2)
			set( ALL_C_FLAGS "${ALL_C_FLAGS} /arch:SSE2")
		else (ZDOOM_USE_SSE2)
			if (MSVC_VERSION GREATER 1699)
				# On Visual C++ 2012 and later SSE2 is the default, so we need to switch it off explicitly
				set( ALL_C_FLAGS "${ALL_C_FLAGS} /arch:IA32")
			endif (MSVC_VERSION GREATER 1699)
		endif (ZDOOM_USE_SSE2)
	endif( CMAKE_SIZEOF_VOID_P MATCHES "4")
	
	# Avoid CRT DLL dependancies in release builds, optionally generate assembly output for checking crash locations.
	option( ZDOOM_GENERATE_ASM "Generate assembly output." OFF )
	if( ZDOOM_GENERATE_ASM )
		set( REL_C_FLAGS "/MT /Oy /Oi /FAcs" )
	else( ZDOOM_GENERATE_ASM )
		set( REL_C_FLAGS "/MT /Oy /Oi" )
	endif( ZDOOM_GENERATE_ASM )

	
	# Debug allocations in debug builds
	set( DEB_C_FLAGS "/D _CRTDBG_MAP_ALLOC /MTd" )

	# Disable warnings for unsecure CRT functions from VC8+
	if( MSVC_VERSION GREATER 1399 )
		set( ALL_C_FLAGS "${ALL_C_FLAGS} /wd4996" )
	endif( MSVC_VERSION GREATER 1399 )
	
	# The CMake configurations set /GR and /MD by default, which conflict with our settings.
	string(REPLACE "/MD " " " CMAKE_CXX_FLAGS_RELEASE ${CMAKE_CXX_FLAGS_RELEASE} )
	string(REPLACE "/MD " " " CMAKE_CXX_FLAGS_MINSIZEREL ${CMAKE_CXX_FLAGS_MINSIZEREL} )
	string(REPLACE "/MD " " " CMAKE_CXX_FLAGS_RELWITHDEBINFO ${CMAKE_CXX_FLAGS_RELWITHDEBINFO} )
	string(REPLACE "/MDd " " " CMAKE_CXX_FLAGS_DEBUG ${CMAKE_CXX_FLAGS_DEBUG} )
	string(REPLACE "/MD " " " CMAKE_C_FLAGS_RELEASE ${CMAKE_C_FLAGS_RELEASE} )
	string(REPLACE "/MD " " " CMAKE_C_FLAGS_MINSIZEREL ${CMAKE_C_FLAGS_MINSIZEREL} )
	string(REPLACE "/MD " " " CMAKE_C_FLAGS_RELWITHDEBINFO ${CMAKE_C_FLAGS_RELWITHDEBINFO} )
	string(REPLACE "/MDd " " " CMAKE_C_FLAGS_DEBUG ${CMAKE_C_FLAGS_DEBUG} )
	string(REPLACE " /GR" " " CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} )
else( MSVC )
	set( REL_LINKER_FLAGS "" )
	set( ALL_C_FLAGS "" )
	set( REL_C_FLAGS "" )
	set( DEB_C_FLAGS "" )
endif( MSVC )

set( CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} ${REL_LINKER_FLAGS}" )
set( CMAKE_EXE_LINKER_FLAGS_MINSIZEREL "${CMAKE_EXE_LINKER_FLAGS_MINSIZEREL} ${REL_LINKER_FLAGS}" )
set( CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO "${CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO} ${REL_LINKER_FLAGS}" )

set( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${ALL_C_FLAGS}" )
set( CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ${REL_C_FLAGS}" )
set( CMAKE_C_FLAGS_MINSIZEREL "${CMAKE_C_FLAGS_MINSIZEREL} ${REL_C_FLAGS}" )
set( CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO} ${REL_C_FLAGS}" )
set( CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} ${DEB_C_FLAGS} -D_DEBUG" )

set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${ALL_C_FLAGS}" )
set( CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} ${REL_C_FLAGS}" )
set( CMAKE_CXX_FLAGS_MINSIZEREL "${CMAKE_CXX_FLAGS_MINSIZEREL} ${REL_C_FLAGS}" )
set( CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} ${REL_C_FLAGS}" )
set( CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} ${DEB_C_FLAGS} -D_DEBUG" )

option(FORCE_INTERNAL_ZLIB "Use internal zlib")
option(FORCE_INTERNAL_JPEG "Use internal jpeg")
option(FORCE_INTERNAL_BZIP2 "Use internal bzip2")
option(FORCE_INTERNAL_GME "Use internal gme" ON)

if( ZLIB_FOUND AND NOT FORCE_INTERNAL_ZLIB )
	message( STATUS "Using system zlib" )
else( ZLIB_FOUND AND NOT FORCE_INTERNAL_ZLIB )
	message( STATUS "Using internal zlib" )
	add_subdirectory( zlib )
	set( ZLIB_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/zlib )
	set( ZLIB_LIBRARIES z )
	set( ZLIB_LIBRARY z )
endif( ZLIB_FOUND AND NOT FORCE_INTERNAL_ZLIB )

if( JPEG_FOUND AND NOT FORCE_INTERNAL_JPEG )
	message( STATUS "Using system jpeg library" )
else( JPEG_FOUND AND NOT FORCE_INTERNAL_JPEG )
	message( STATUS "Using internal jpeg library" )
	add_subdirectory( jpeg-6b )
	set( JPEG_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/jpeg-6b )
	set( JPEG_LIBRARIES jpeg )
	set( JPEG_LIBRARY jpeg )
endif( JPEG_FOUND AND NOT FORCE_INTERNAL_JPEG )

if( BZIP2_FOUND AND NOT FORCE_INTERNAL_BZIP2 )
	message( STATUS "Using system bzip2 library" )
else( BZIP2_FOUND AND NOT FORCE_INTERNAL_BZIP2 )
	message( STATUS "Using internal bzip2 library" )
	add_subdirectory( bzip2 )
	set( BZIP2_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/bzip2" )
	set( BZIP2_LIBRARIES bz2 )
	set( BZIP2_LIBRARY bz2 )
endif( BZIP2_FOUND AND NOT FORCE_INTERNAL_BZIP2 )

if( GME_FOUND AND NOT FORCE_INTERNAL_GME )
	message( STATUS "Using system gme library ${GME_INCLUDE_DIR}" )
else( GME_FOUND AND NOT FORCE_INTERNAL_GME )
	message( STATUS "Using internal gme library" )
	add_subdirectory( game-music-emu )
	set( GME_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/game-music-emu" )
	set( GME_LIBRARIES gme )
endif( GME_FOUND AND NOT FORCE_INTERNAL_GME )

set( LZMA_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/lzma/C" )

if( NOT CMAKE_CROSSCOMPILING )
	if( NOT CROSS_EXPORTS )
		set( CROSS_EXPORTS "" )
	endif( NOT CROSS_EXPORTS )
endif( NOT CMAKE_CROSSCOMPILING )

add_subdirectory( lzma )
add_subdirectory( tools )
add_subdirectory( dumb )
add_subdirectory( gdtoa )
add_subdirectory( wadsrc )
add_subdirectory( wadsrc_bm )
add_subdirectory( wadsrc_lights )
add_subdirectory( src )

if( NOT WIN32 AND NOT APPLE )
	add_subdirectory( output_sdl )
endif( NOT WIN32 AND NOT APPLE )

if( NOT CMAKE_CROSSCOMPILING )
	export(TARGETS ${CROSS_EXPORTS} FILE "${CMAKE_BINARY_DIR}/ImportExecutables.cmake" )
endif( NOT CMAKE_CROSSCOMPILING )

# INSTALL

install(DIRECTORY docs DESTINATION docs)

# Optionally install a wad file to simplify install->run
set(ZDOOM_DEFAULT_WAD "" CACHE FILEPATH "WAD file to install with zdoom")
if (ZDOOM_DEFAULT_WAD)
    install(FILES ${ZDOOM_DEFAULT_WAD} DESTINATION ".")
endif()

# Install Windows Rift shortcuts
if(WIN32)
    file(GLOB WIN_SHORTCUTS "gz3doom/GZ3DoomRiftMonitor1.bat")
    install(PROGRAMS ${WIN_SHORTCUTS} DESTINATION ".")
endif()

# PACKAGE installer
set(INSTALL_PROGRAM_NAME GZ3Doom)
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${INSTALL_PROGRAM_NAME} game in stereoscopic 3D")
set(CPACK_PACKAGE_VENDOR "zdoom.org")
set(CPACK_PACKAGE_EXECUTABLES ${ZDOOM_EXE_NAME} ${INSTALL_PROGRAM_NAME})
# TODO - read actual zdoom version from somewheare
set(CPACK_PACKAGE_VERSION_MAJOR "1")
set(CPACK_PACKAGE_VERSION_MINOR "8")
set(CPACK_PACKAGE_VERSION_PATCH "10_e")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/docs/LICENSE.txt")
set(CPACK_PACKAGE_NAME ${INSTALL_PROGRAM_NAME})
set(COMBINED_VERSION "v${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}")
set(NAME_WITH_VERSION "${INSTALL_PROGRAM_NAME}_${COMBINED_VERSION}")

if (WIN32)
	set(CPACK_GENERATOR NSIS)
	# Both are needed, to set the installer icon
	set(CPACK_NSIS_MUI_UNIICON "${CMAKE_SOURCE_DIR}/src/win32\\\\icon1.ico")
	set(CPACK_NSIS_MUI_ICON "${CMAKE_SOURCE_DIR}/src/win32\\\\icon1.ico")
	#
	set(CPACK_NSIS_MUI_FINISHPAGE_RUN ${ZDOOM_EXE_NAME})
	set(CPACK_NSIS_EXECUTABLES_DIRECTORY ".")
	if (MSVC10)
		install(PROGRAMS installer/msvc10/vcredist_x86.exe DESTINATION ".")
		set(CPACK_NSIS_EXTRA_INSTALL_COMMANDS "ExecWait '\\\"$INSTDIR\\\\vcredist_x86.exe\\\" /passive /norestart' ")
	endif()
	if (MSVC14)
		install(PROGRAMS installer/msvc14/vc_redist.x86.exe DESTINATION ".")
		set(CPACK_NSIS_EXTRA_INSTALL_COMMANDS "ExecWait '\\\"$INSTDIR\\\\vc_redist.x86.exe\\\" /passive /norestart' ")
	endif()
    set(CPACK_NSIS_MENU_LINKS
        "GZ3DoomRiftMonitor1.bat" "GZ3Doom on Oculus Rift"
    )
endif()

if (APPLE)
	set(CPACK_GENERATOR Bundle)
	set(CPACK_PACKAGE_FILE_NAME "${NAME_WITH_VERSION}_mac")
	set(CPACK_BUNDLE_NAME ${INSTALL_PROGRAM_NAME})
        configure_file(${CMAKE_SOURCE_DIR}/installer/mac/Info.plist.in ${CMAKE_BINARY_DIR}/Info.plist @ONLY)
        set(CPACK_BUNDLE_PLIST ${CMAKE_BINARY_DIR}/Info.plist)
        set(CPACK_BUNDLE_STARTUP_COMMAND "${CMAKE_SOURCE_DIR}/installer/mac/GZ3Doom")
        set(CPACK_BUNDLE_ICON "${CMAKE_SOURCE_DIR}/installer/mac/doomguy1.icns")
endif()

include(CPack)
