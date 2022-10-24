CMAKE_MINIMUM_REQUIRED(VERSION 3.7)

SET(VERSION_EXTERNAL "0.5.1")
SET(VERSION_PACKAGE "1")

PROJECT("bam")

SET(PROJECT_VERSION "${VERSION_EXTERNAL}.${VERSION_PACKAGE}")

INCLUDE(ExternalProject)

# Filter the jonchki configuration.
CONFIGURE_FILE(installer/bam.xml
               ${CMAKE_BINARY_DIR}/bam-${PROJECT_VERSION}.xml
               @ONLY)

#----------------------------------------------------------------------------
#
# Build the project.
#
IF((${BUILDCFG_ONLY_JONCHKI_CFG} STREQUAL "OFF"))
	# Select the LUA version.
	INCLUDE(${CMAKE_HOME_DIRECTORY}/cmake/muhkuh_lua.cmake)

	SET(PRJ_TXT2C_CMAKE_ARGS "")
	LIST(APPEND PRJ_TXT2C_CMAKE_ARGS "-DCMAKE_BUILD_TYPE=Release")
	LIST(APPEND PRJ_TXT2C_CMAKE_ARGS "-DBUILDCFG_ONLY_TXT2C=ON")
	LIST(APPEND PRJ_TXT2C_CMAKE_ARGS "-DBUILDCFG_INSTALL_TOOLS=${CMAKE_CURRENT_BINARY_DIR}/install/tools")

	ExternalProject_Add(TARGET_txt2c
	                    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/bam
	                    URL ${CMAKE_CURRENT_SOURCE_DIR}/bam-b86962a.zip
	                    URL_HASH SHA1=396fc6a092cad00331cda780f7dcfdc89f084a30
	                    PATCH_COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/copy_over ${CMAKE_CURRENT_BINARY_DIR}/bam/src/TARGET_txt2c
	                    CMAKE_ARGS ${PRJ_TXT2C_CMAKE_ARGS}
	                    INSTALL_COMMAND make install
	)


	SET(PRJ_BAM_CMAKE_ARGS "")
	LIST(APPEND PRJ_BAM_CMAKE_ARGS "-DCMAKE_BUILD_TYPE=Release")
	LIST(APPEND PRJ_BAM_CMAKE_ARGS "-DLUA_LIBRARY=${LUA_LIBRARIES}")
	LIST(APPEND PRJ_BAM_CMAKE_ARGS "-DLUA_INCLUDE_DIR=${LUA_INCLUDE_DIR}")
	LIST(APPEND PRJ_BAM_CMAKE_ARGS "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}")
	LIST(APPEND PRJ_BAM_CMAKE_ARGS "-DBUILDCFG_ONLY_TXT2C=OFF")
	LIST(APPEND PRJ_BAM_CMAKE_ARGS "-DBUILDCFG_TXT2C_TOOL=${CMAKE_CURRENT_BINARY_DIR}/install/tools/txt2c")
	LIST(APPEND PRJ_BAM_CMAKE_ARGS "-DBUILDCFG_INSTALL_EXE=${CMAKE_CURRENT_BINARY_DIR}/install/bam")
	LIST(APPEND PRJ_BAM_CMAKE_ARGS "-DCMAKE_INSTALL_PREFIX=''")

	ExternalProject_Add(TARGET_bam
	                    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/bam
	                    URL ${CMAKE_CURRENT_SOURCE_DIR}/bam-b86962a.zip
	                    URL_HASH SHA1=396fc6a092cad00331cda780f7dcfdc89f084a30
	                    PATCH_COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/copy_over ${CMAKE_CURRENT_BINARY_DIR}/bam/src/TARGET_bam
	                    CMAKE_ARGS ${PRJ_BAM_CMAKE_ARGS}
	                    INSTALL_COMMAND make install
	                    DEPENDS TARGET_txt2c
	)

	IF((CMAKE_SYSTEM_NAME STREQUAL "Windows") AND (${CMAKE_COMPILER_IS_GNUCC}))
		# Here are the MinGW specific tests.
		ADD_TEST(NAME bam_MinGW_DLL_dependencies
		         COMMAND "${PYTHON_INTERPRETER}" ${CMAKE_SOURCE_DIR}/cmake/tests/mingw_dll_dependencies.py -u lua5.1 -u lua5.2 -u lua5.3 -u lua5.4 ${CMAKE_CURRENT_BINARY_DIR}/install/bam/bam.exe)
	ENDIF((CMAKE_SYSTEM_NAME STREQUAL "Windows") AND (${CMAKE_COMPILER_IS_GNUCC}))


	#----------------------------------------------------------------------------
	#
	# Build a package for the selected platform.
	#
	SET(CPACK_OUTPUT_FILE_PREFIX ${CMAKE_BINARY_DIR}/../../repository/com/github/matricks/bam/${PROJECT_VERSION})
	SET(CPACK_INCLUDE_TOPLEVEL_DIRECTORY FALSE)
	SET(CPACK_PACKAGE_FILE_NAME "bam-${PROJECT_VERSION}-${JONCHKI_PLATFORM}")

	SET(CPACK_GENERATOR "TXZ")
	INCLUDE(CPack)

	IF(${JONCHKI_PLATFORM_DIST_ID} STREQUAL "windows")
		# Package a windows build.

		CONFIGURE_FILE(installer/install_windows.lua
		               ${CMAKE_CURRENT_BINARY_DIR}/package/install.lua
		               @ONLY)

		INSTALL(FILES ${CMAKE_CURRENT_BINARY_DIR}/install/bam/bam.exe
		        DESTINATION ${INSTALL_DIR_EXECUTABLES}
		        PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_WRITE GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
	ELSE(${JONCHKI_PLATFORM_DIST_ID} STREQUAL "windows")
		# Package a linux build.

		CONFIGURE_FILE(installer/install_linux.lua
		               ${CMAKE_CURRENT_BINARY_DIR}/package/install.lua
		               @ONLY)

		INSTALL(FILES ${CMAKE_CURRENT_BINARY_DIR}/install/bam/bam
		        DESTINATION ${INSTALL_DIR_EXECUTABLES}
		        PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_WRITE GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
	ENDIF(${JONCHKI_PLATFORM_DIST_ID} STREQUAL "windows")

	INSTALL(FILES ${CMAKE_CURRENT_BINARY_DIR}/package/install.lua
	        DESTINATION .)

	CONFIGURE_FILE(installer/bam.xml
	               ${CPACK_OUTPUT_FILE_PREFIX}/bam-${PROJECT_VERSION}-${JONCHKI_PLATFORM}.xml
	               @ONLY)


	#-----------------------------------------------------------------------------
	#
	# Create a custom target named "pack".
	# It builds the package and creates the hashes.
	#
	ADD_CUSTOM_TARGET(pack
	                  COMMAND ${CMAKE_MAKE_PROGRAM}
	                  COMMAND ${CMAKE_MAKE_PROGRAM} package
	                  COMMAND "${PYTHON_INTERPRETER}" ${CMAKE_HOME_DIRECTORY}/cmake/tools/generate_hash.py ${CPACK_OUTPUT_FILE_PREFIX}/bam-${PROJECT_VERSION}-${JONCHKI_PLATFORM}.xml
	                  COMMAND "${PYTHON_INTERPRETER}" ${CMAKE_HOME_DIRECTORY}/cmake/tools/generate_hash.py ${CPACK_OUTPUT_FILE_PREFIX}/bam-${PROJECT_VERSION}-${JONCHKI_PLATFORM}.tar.xz
	                  COMMENT "Pack the artifact and generate the hashes...")


	#-----------------------------------------------------------------------------
	#
	# Filter the pom.xml file.
	#
	SET(POM_ARTIFACT_ID  "bam")
	SET(POM_GROUP_ID     "com.github.matricks")
	SET(POM_VERSION      "${PROJECT_VERSION}")
	SET(POM_PACKAGING    "tar.xz")
	CONFIGURE_FILE(installer/pom.xml
	               ${CPACK_OUTPUT_FILE_PREFIX}/bam-${PROJECT_VERSION}.pom
	               @ONLY)
ENDIF((${BUILDCFG_ONLY_JONCHKI_CFG} STREQUAL "OFF"))
