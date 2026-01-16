cmake_minimum_required(VERSION 3.10)

project(SlamtecCHOP)

set(CMAKE_CXX_STANDARD 17)

# Platform-specific C++ flags
if (APPLE)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17 -stdlib=libc++")
elseif(WIN32)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /std:c++17")
endif()

# Output directory
set(OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/plugin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${OUTPUT_DIRECTORY})

# Define the plugin name
set(PLUGIN_NAME Slamtec)

# RPLiDAR SDK path
set(RPLIDAR_SDK_PATH ${CMAKE_CURRENT_SOURCE_DIR}/rplidar_sdk)

# ============================================================================
# Build RPLidar SDK from source
# ============================================================================

# SDK include directories
set(SDK_INCLUDE_DIRS
	${RPLIDAR_SDK_PATH}/sdk/include
	${RPLIDAR_SDK_PATH}/sdk/src
	${RPLIDAR_SDK_PATH}/sdk/src/hal
	${RPLIDAR_SDK_PATH}/sdk/src/dataunpacker
	${RPLIDAR_SDK_PATH}/sdk/src/dataunpacker/unpacker
)

# Core SDK source files
set(SDK_CORE_SOURCES
	${RPLIDAR_SDK_PATH}/sdk/src/sl_lidar_driver.cpp
	${RPLIDAR_SDK_PATH}/sdk/src/sl_crc.cpp
	${RPLIDAR_SDK_PATH}/sdk/src/sl_serial_channel.cpp
	${RPLIDAR_SDK_PATH}/sdk/src/sl_tcp_channel.cpp
	${RPLIDAR_SDK_PATH}/sdk/src/sl_udp_channel.cpp
	${RPLIDAR_SDK_PATH}/sdk/src/sl_lidarprotocol_codec.cpp
	${RPLIDAR_SDK_PATH}/sdk/src/sl_async_transceiver.cpp
	${RPLIDAR_SDK_PATH}/sdk/src/hal/thread.cpp
)

# Dataunpacker sources
set(SDK_DATAUNPACKER_SOURCES
	${RPLIDAR_SDK_PATH}/sdk/src/dataunpacker/dataunpacker.cpp
	${RPLIDAR_SDK_PATH}/sdk/src/dataunpacker/unpacker/handler_capsules.cpp
	${RPLIDAR_SDK_PATH}/sdk/src/dataunpacker/unpacker/handler_hqnode.cpp
	${RPLIDAR_SDK_PATH}/sdk/src/dataunpacker/unpacker/handler_normalnode.cpp
)

# Platform-specific sources
if (WIN32)
	set(SDK_PLATFORM_SOURCES
		${RPLIDAR_SDK_PATH}/sdk/src/arch/win32/net_serial.cpp
		${RPLIDAR_SDK_PATH}/sdk/src/arch/win32/net_socket.cpp
		${RPLIDAR_SDK_PATH}/sdk/src/arch/win32/timer.cpp
	)
elseif(APPLE)
	set(SDK_PLATFORM_SOURCES
		${RPLIDAR_SDK_PATH}/sdk/src/arch/macOS/net_serial.cpp
		${RPLIDAR_SDK_PATH}/sdk/src/arch/macOS/net_socket.cpp
		${RPLIDAR_SDK_PATH}/sdk/src/arch/macOS/timer.cpp
	)
elseif(UNIX)
	set(SDK_PLATFORM_SOURCES
		${RPLIDAR_SDK_PATH}/sdk/src/arch/linux/net_serial.cpp
		${RPLIDAR_SDK_PATH}/sdk/src/arch/linux/net_socket.cpp
		${RPLIDAR_SDK_PATH}/sdk/src/arch/linux/timer.cpp
	)
endif()

# Combine all SDK sources
set(SDK_ALL_SOURCES
	${SDK_CORE_SOURCES}
	${SDK_DATAUNPACKER_SOURCES}
	${SDK_PLATFORM_SOURCES}
)

# Create SDK static library
add_library(sl_lidar_sdk STATIC ${SDK_ALL_SOURCES})

target_include_directories(sl_lidar_sdk PUBLIC ${SDK_INCLUDE_DIRS})

# Platform-specific SDK settings
if (WIN32)
	target_compile_definitions(sl_lidar_sdk PRIVATE _WIN32)
	# Windows system libraries will be linked at plugin level
elseif(APPLE)
	target_compile_definitions(sl_lidar_sdk PRIVATE __APPLE__ _MACOS)
endif()

# ============================================================================
# Build TouchDesigner Plugin
# ============================================================================

# Plugin source files
set(PLUGIN_SOURCES
	SlamtecCHOP.cpp
	Parameters.cpp
	drvlogic/RPLidarDevice.cpp
)

# TouchDesigner headers
include_directories(.)

# Create plugin as MODULE (shared library)
add_library(${PLUGIN_NAME} MODULE ${PLUGIN_SOURCES})

set_target_properties(${PLUGIN_NAME} PROPERTIES LINKER_LANGUAGE CXX)

# Platform-specific plugin configuration
if (WIN32)
	# Windows: Output as .dll
	set_target_properties(${PLUGIN_NAME} PROPERTIES PREFIX "")
	set_target_properties(${PLUGIN_NAME} PROPERTIES SUFFIX ".dll")
	
	# Set output directory for Windows (MODULE targets use RUNTIME_OUTPUT_DIRECTORY)
	# Visual Studio generator creates config subdirectories, so we need to set both
	set_target_properties(${PLUGIN_NAME} PROPERTIES
		RUNTIME_OUTPUT_DIRECTORY ${OUTPUT_DIRECTORY}
		RUNTIME_OUTPUT_DIRECTORY_DEBUG ${OUTPUT_DIRECTORY}
		RUNTIME_OUTPUT_DIRECTORY_RELEASE ${OUTPUT_DIRECTORY}
		RUNTIME_OUTPUT_DIRECTORY_MINSIZEREL ${OUTPUT_DIRECTORY}
		RUNTIME_OUTPUT_DIRECTORY_RELWITHDEBINFO ${OUTPUT_DIRECTORY}
		# Prevent Visual Studio from creating config subdirectories
		VS_DEBUGGER_WORKING_DIRECTORY ${OUTPUT_DIRECTORY}
	)
	
	# Windows system libraries
	target_link_libraries(${PLUGIN_NAME} PRIVATE 
		setupapi
		ws2_32
	)
	
	# Windows-specific compile definitions
	target_compile_definitions(${PLUGIN_NAME} PRIVATE
		WIN32
		_WINDOWS
		_USRDLL
		CPLUSPLUSCHOPEXAMPLE_EXPORTS
	)
elseif(APPLE)
	# macOS: Output as .plugin bundle
	set_target_properties(${PLUGIN_NAME} PROPERTIES PREFIX "")
	set_target_properties(${PLUGIN_NAME} PROPERTIES BUNDLE TRUE)
	set_target_properties(${PLUGIN_NAME} PROPERTIES BUNDLE_EXTENSION "plugin")
	
	# Set bundle output location
	set_target_properties(${PLUGIN_NAME} PROPERTIES MACOSX_BUNDLE TRUE)
	
	# Set Info.plist if it exists, otherwise CMake will generate a default one
	set(INFO_PLIST_PATH ${CMAKE_CURRENT_SOURCE_DIR}/plugin/Slamtec.plugin/Contents/Info.plist)
	if(EXISTS ${INFO_PLIST_PATH})
		set_target_properties(${PLUGIN_NAME} PROPERTIES MACOSX_BUNDLE_INFO_PLIST ${INFO_PLIST_PATH})
	endif()
endif()

# Link SDK to plugin
target_link_libraries(${PLUGIN_NAME} PRIVATE sl_lidar_sdk)

# Include SDK headers in plugin
target_include_directories(${PLUGIN_NAME} PRIVATE ${SDK_INCLUDE_DIRS})