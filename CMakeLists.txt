cmake_minimum_required(VERSION 3.2)

if(NOT BOARD)
	message(FATAL_ERROR "BOARD must be set (eg stm32f769i-disco)")
endif()

if(NOT CONFIG_TYPE)
	# default to nsh if not specified
	set(CONFIG_TYPE "nsh")
endif()

project(NuttX LANGUAGES ASM C CXX)
message(STATUS "NuttX: ${BOARD}/${CONFIG_TYPE}")

set(NUTTX_DIR ${CMAKE_CURRENT_SOURCE_DIR})

# .config
add_custom_command(
	OUTPUT ${NUTTX_DIR}/.config
	COMMAND tools/configure.sh -l ${BOARD}/${CONFIG_TYPE}
	DEPENDS
		${NUTTX_DIR}/configs/${BOARD}/${CONFIG_TYPE}/defconfig
	WORKING_DIRECTORY ${NUTTX_DIR}
	)

# .version
add_custom_command(
	OUTPUT ${NUTTX_DIR}/.version
	COMMAND tools/version.sh -v 0.0 -b 0 .version
	DEPENDS
		${NUTTX_DIR}/configs/${BOARD}/${CONFIG_TYPE}/defconfig
	WORKING_DIRECTORY ${NUTTX_DIR}
	)


include(ExternalProject)
ExternalProject_Add(nuttx_tools
	SOURCE_DIR ${CMAKE_SOURCE_DIR}/tools
	INSTALL_DIR ${CMAKE_BINARY_DIR}
	CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}
	)

# mkconfig
add_custom_command(OUTPUT ${NUTTX_DIR}/include/nuttx/config.h
	COMMAND ${CMAKE_BINARY_DIR}/bin/mkconfig ${NUTTX_DIR} > ${NUTTX_DIR}/include/nuttx/config.h
	DEPENDS nuttx_tools ${NUTTX_DIR}/.config
	WORKING_DIRECTORY ${NUTTX_DIR}
	)
add_custom_target(nuttx_mkconfig DEPENDS ${NUTTX_DIR}/include/nuttx/config.h)

# mkversion
add_custom_command(OUTPUT ${NUTTX_DIR}/include/nuttx/version.h
	COMMAND ${CMAKE_BINARY_DIR}/bin/mkversion ${NUTTX_DIR} > ${NUTTX_DIR}/include/nuttx/version.h
	DEPENDS nuttx_tools ${NUTTX_DIR}/.version
	WORKING_DIRECTORY ${NUTTX_DIR}
	)
add_custom_target(nuttx_mkversion DEPENDS ${NUTTX_DIR}/include/nuttx/version.h)



add_custom_command(
	OUTPUT
		${NUTTX_DIR}/include/arch
		${NUTTX_DIR}/include/arch/board
		${NUTTX_DIR}/include/arch/chip
		${NUTTX_DIR}/arch/arm/src/board
		${NUTTX_DIR}/arch/arm/src/chip
	COMMAND ${CMAKE_COMMAND} -E create_symlink ${NUTTX_DIR}/arch/arm/include ${NUTTX_DIR}/include/arch
	COMMAND ${CMAKE_COMMAND} -E create_symlink ${NUTTX_DIR}/configs/${BOARD}/${CONFIG_TYPE}/include ${NUTTX_DIR}/include/arch/board
	COMMAND ${CMAKE_COMMAND} -E create_symlink ${NUTTX_DIR}/arch/arm/include/stm32f7 ${NUTTX_DIR}/include/arch/chip
	COMMAND ${CMAKE_COMMAND} -E create_symlink ${NUTTX_DIR}/configs/${BOARD}/${CONFIG_TYPE}/src ${NUTTX_DIR}/arch/arm/src/board
	COMMAND ${CMAKE_COMMAND} -E create_symlink ${NUTTX_DIR}/arch/arm/src/stm32f7 ${NUTTX_DIR}/arch/arm/src/chip
)
add_custom_target(nuttx_symlinks DEPENDS ${NUTTX_DIR}/include/arch)

# oldconfig helper
add_custom_target(oldconfig
	COMMAND make --no-print-directory --silent -C ${NUTTX_DIR} CONFIG_ARCH_BOARD=${BOARD} oldconfig
	COMMAND cp ${NUTTX_DIR}/.config ${NUTTX_CONFIG_DIR}/${BOARD}/${nuttx_config_type}/defconfig
	COMMAND ${PX4_SOURCE_DIR}/platforms/nuttx/NuttX/tools/nuttx_defconf_tool.sh ${NUTTX_CONFIG_DIR}/${BOARD}/${nuttx_config_type}/defconfig
	DEPENDS nuttx_configure
	WORKING_DIRECTORY ${NUTTX_DIR}
	COMMENT "Running NuttX make oldconfig for ${BOARD} with ${nuttx_config_type}"
	USES_TERMINAL
	)

# menuconfig helper
add_custom_target(menuconfig
	COMMAND make --no-print-directory --silent -C ${NUTTX_DIR} CONFIG_ARCH_BOARD=${BOARD} menuconfig
	COMMAND cp ${NUTTX_DIR}/.config ${NUTTX_CONFIG_DIR}/${BOARD}/${nuttx_config_type}/defconfig
	COMMAND cd ${PX4_SOURCE_DIR} && platforms/nuttx/NuttX/tools/nuttx_defconf_tool.sh ${NUTTX_CONFIG_DIR}/${BOARD}/${nuttx_config_type}/defconfig
	DEPENDS nuttx_configure
	WORKING_DIRECTORY ${NUTTX_DIR}
	COMMENT "Running NuttX make menuconfig for ${BOARD} with ${nuttx_config_type}"
	USES_TERMINAL
	)

# qconfig helper
add_custom_target(qconfig
	COMMAND make --no-print-directory --silent -C ${NUTTX_DIR} CONFIG_ARCH_BOARD=${BOARD} qconfig
	COMMAND cp .config ${NUTTX_CONFIG_DIR}/${BOARD}/${nuttx_config_type}/defconfig
	DEPENDS nuttx_configure
	WORKING_DIRECTORY ${NUTTX_DIR}
	COMMENT "Running NuttX make qconfig for ${BOARD} with ${nuttx_config_type}"
	USES_TERMINAL
	)
