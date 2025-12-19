cmake_minimum_required(VERSION 3.10)
set(PROJECT_NAME rosbridge_client_cpp)
project(${PROJECT_NAME})

message(STATUS "Ready to build ${PROJECT_NAME}")

# Find required packages
find_package(PkgConfig REQUIRED)
# Prefer static Boost libraries for deployment compatibility
# This ensures the binary doesn't depend on specific Boost .so versions on the target system
set(Boost_USE_STATIC_LIBS ON)
set(Boost_USE_STATIC_RUNTIME OFF)  # Use dynamic runtime, but static Boost libs
set(Boost_USE_MULTITHREADED ON)
find_package(Boost REQUIRED COMPONENTS system)
find_package(Threads REQUIRED)

# Verify Boost linking (for debugging)
if(Boost_FOUND)
    message(STATUS "Boost version: ${Boost_VERSION}")
    message(STATUS "Boost libraries: ${Boost_LIBRARIES}")
    message(STATUS "Boost system library: ${Boost_SYSTEM_LIBRARY}")
endif()

# BSON support has been removed - using JSON mode only

if(NOT WIN32)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -Wall")
endif()

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Libraries
set(LIBS ${LIBS} ${CMAKE_THREAD_LIBS_INIT})
set(LIBS ${LIBS} ${Boost_LIBRARIES})

# Example executables (only built when this is the main project)
# These compile sources directly for demonstration purposes
if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
	#WebSocket Client
	ADD_EXECUTABLE( websocket_client
		src/client/websocket_client.cpp 
		src/client/socket_websocket_connection.cpp 
		src/ros_bridge.cpp 
		src/ros_topic.cpp 
		src/ros_service.cpp 
		src/ros_tf_broadcaster.cpp 
	)
	target_include_directories(websocket_client PRIVATE include ${Boost_INCLUDE_DIRS})
	target_compile_definitions(websocket_client PRIVATE RAPIDJSON_HAS_STDSTRING=1)

	# Alternative TCP client (for ROS1 compatibility)
	ADD_EXECUTABLE( tcp_client
		src/client/tcp_client.cpp 
		src/client/socket_tcp_connection.cpp 
		src/ros_bridge.cpp 
		src/ros_topic.cpp 
		src/ros_service.cpp 
		src/ros_tf_broadcaster.cpp 
	)
	target_include_directories(tcp_client PRIVATE include ${Boost_INCLUDE_DIRS})
	target_compile_definitions(tcp_client PRIVATE RAPIDJSON_HAS_STDSTRING=1)

	# Comprehensive demo client showing multiple features
	# Links against the rosbridge_client_cpp library instead of compiling all sources
	ADD_EXECUTABLE( demo_client
		src/client/demo_client.cpp 
	)
endif()

# Main library target
ADD_LIBRARY( rosbridge_client_cpp STATIC
	src/ros_bridge.cpp 
	src/ros_topic.cpp 
	src/ros_service.cpp 
	src/ros_tf_broadcaster.cpp 
	src/client/socket_websocket_connection.cpp 
	src/client/socket_tcp_connection.cpp 
)

# Configure the library target with proper include directories and definitions
target_include_directories(rosbridge_client_cpp
	PUBLIC
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
		$<INSTALL_INTERFACE:include>
	PRIVATE
		${Boost_INCLUDE_DIRS}
)

target_compile_definitions(rosbridge_client_cpp
	PUBLIC
		RAPIDJSON_HAS_STDSTRING=1
)

target_link_libraries(rosbridge_client_cpp
	PRIVATE
		${CMAKE_THREAD_LIBS_INIT}
		${Boost_LIBRARIES}
)

# Copy to project lib directory (main project /lib, not install folder)
add_custom_command(TARGET rosbridge_client_cpp POST_BUILD
	COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_SOURCE_DIR}/lib
	COMMAND ${CMAKE_COMMAND} -E copy_if_different
		$<TARGET_FILE:rosbridge_client_cpp>
		${CMAKE_SOURCE_DIR}/lib/
	COMMENT "Copying rosbridge_client_cpp library to lib directory"
)

# Install the library to install prefix (needed for installed executables)
# install(TARGETS rosbridge_client_cpp
# 	LIBRARY DESTINATION lib
# 	ARCHIVE DESTINATION lib
# 	RUNTIME DESTINATION bin
# )

# Link example executables (only if they were built)
if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
	target_link_libraries(websocket_client ${LIBS})
	target_link_libraries(tcp_client ${LIBS})
	target_link_libraries(demo_client rosbridge_client_cpp ${LIBS})
endif()