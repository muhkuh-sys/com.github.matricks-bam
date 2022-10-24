cmake_minimum_required(VERSION 3.7)

PROJECT(
	"com.github.matricks-bam"
	VERSION ${VERSION}
)


# We need to build a little tool for the host: txt2c
# It is used in the build process, not on the target system.
# This means it must not be built with the toolchain for the target, but with the compiler for the build host.
#
# Example: We are building on a desktop PC (x86_64) for ARM64.
#          If txt2c would be build with the toolchain settings, it would end up as ARM64 code.
#          Trying to run it in this form on the x86_64 PC would not work.
#
# Unfortunately CMake does not support multiple toolchains in the same build (see https://discourse.cmake.org/t/using-multiple-toolchains-in-the-same-project/1505 ).
# So it is not possible to build txt2c for the host and the rest for the target system.
#
# Here is a workaround: Run cmake twice.
# The first time, the variable "BUILDCFG_ONLY_TXT2C" is set to "ON" and no toolchain file is set. Now "txt2c" is build.
# The second time, the variable "BUILDCFG_ONLY_TXT2C" is set to "OFF" and the toolchain file is set for the target. Now "bam" is build.
IF((${BUILDCFG_ONLY_TXT2C} STREQUAL "ON"))
	ADD_EXECUTABLE(TARGET_txt2cexe
	               src/tools/txt2c.c)

	SET_TARGET_PROPERTIES(TARGET_txt2cexe PROPERTIES
	                      OUTPUT_NAME txt2c)

	INSTALL(TARGETS TARGET_txt2cexe
	        RUNTIME DESTINATION ${BUILDCFG_INSTALL_TOOLS})

ELSE((${BUILDCFG_ONLY_TXT2C} STREQUAL "ON"))
	SET(CMAKE_SKIP_RPATH FALSE)
	SET(CMAKE_SKIP_BUILD_RPATH FALSE)
	SET(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
	SET(CMAKE_BUILD_RPATH "$ORIGIN")
	SET(CMAKE_INSTALL_RPATH "$ORIGIN")

	ADD_CUSTOM_TARGET(TARGET_luatxt ALL
	                  COMMAND "${BUILDCFG_TXT2C_TOOL}" src/base.lua src/tools.lua src/driver_gcc.lua src/driver_clang.lua src/driver_cl.lua src/driver_solstudio.lua src/driver_xlc.lua > ${CMAKE_CURRENT_BINARY_DIR}/internal_base.h
	                  WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
	                  COMMENT "Convert LUA sources to C."
	                  BYPRODUCTS ${CMAKE_CURRENT_BINARY_DIR}/txt2c/internal_base.h
	)

	#-----------------------------------------------------------------------------
	#
	# Build the bam executable.
	#

	SET(BAM_SOURCES
		src/cache.c
		src/context.c
		src/dep_cpp2.c
		src/dep_cpp.c
		src/dep_search.c
		src/luafuncs.c
		src/main.c
		src/mem.c
		src/node.c
		src/path.c
		src/statcache.c
		src/support.c
		src/verify.c
	)


	ADD_EXECUTABLE(TARGET_bamexe
	               ${BAM_SOURCES})
	TARGET_LINK_LIBRARIES(TARGET_bamexe
	                      PRIVATE ${LUA_LIBRARY} pthread)
	TARGET_INCLUDE_DIRECTORIES(TARGET_bamexe
	                           PRIVATE ${LUA_INCLUDE_DIR} ${CMAKE_CURRENT_BINARY_DIR})
	SET_TARGET_PROPERTIES(TARGET_bamexe PROPERTIES
	                      OUTPUT_NAME bam)
	ADD_DEPENDENCIES(TARGET_bamexe TARGET_luatxt)

	# On MinGW link all compiler libraries static.
	IF((${CMAKE_SYSTEM_NAME} STREQUAL "Windows") AND (${CMAKE_COMPILER_IS_GNUCC}))
		SET_PROPERTY(TARGET TARGET_bamexe PROPERTY LINK_FLAGS "-static-libgcc -static-libstdc++")
	ENDIF((${CMAKE_SYSTEM_NAME} STREQUAL "Windows") AND (${CMAKE_COMPILER_IS_GNUCC}))

	INSTALL(TARGETS TARGET_bamexe
	        RUNTIME DESTINATION ${BUILDCFG_INSTALL_EXE})
ENDIF((${BUILDCFG_ONLY_TXT2C} STREQUAL "ON"))
