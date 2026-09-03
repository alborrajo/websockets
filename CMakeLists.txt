
cmake_minimum_required(VERSION 3.16)

set(PRE_CONFIGURE_FILE "${CMAKE_CURRENT_SOURCE_DIR}/cmake/git.h.in")
set(POST_CONFIGURE_FILE "${CMAKE_CURRENT_BINARY_DIR}/git.h")
include (${CMAKE_CURRENT_SOURCE_DIR}/cmake/git_watcher.cmake)
include (${CMAKE_CURRENT_SOURCE_DIR}/cmake/functions.cmake)

option(USE_BUNDLED_LWS "Use the LWS library version bundled in this project as opposed to an external one provided by the client project" ON)

if(USE_BUNDLED_LWS)
add_subdirectory(libwebsockets)
endif()

# Use the shared target if available. This is done in case we're not using the bundled LWS.
# For example, when using vcpkg, depending on the triplet, lws can be available as static or shared.
if(TARGET websockets_shared)
set(LIBWEBSOCKETS websockets_shared)
else()
set(LIBWEBSOCKETS websockets)
endif()

project(WebSockets)
SET(CMAKE_CONFIGURATION_TYPES "Debug;Release;MinSizeRel;RelWithDebInfo" CACHE STRING "" FORCE)


set (demo_SRC
	WebSockets.cpp
	main.cpp
	)
set (demo_H
	WebSockets.cpp
	)
add_executable(demo 
	${demo_SRC}
	${demo_H}
	)
target_link_libraries(demo PRIVATE ${LIBWEBSOCKETS})
target_link_libraries(demo PRIVATE
	-Wl,--start-group
	${LIBWEBSOCKETS}
	pthread
	-Wl,--end-group
	)


set (fxopen_SRC
	WebSockets.cpp
	fxopen.cpp
	)
add_executable(fxopen 
	${fxopen_SRC}
	${fxopen_H}
	)
target_link_libraries(fxopen PRIVATE ${LIBWEBSOCKETS})
target_link_libraries(fxopen PRIVATE
	-Wl,--start-group
	${LIBWEBSOCKETS}
	pthread
	ssl 
	crypto 
	uuid
	jsoncpp
	-Wl,--end-group
	)


add_library(WebSockets WebSockets.cpp WebSockets.h)
target_link_libraries(WebSockets PUBLIC ${LIBWEBSOCKETS})
target_link_libraries(WebSockets PUBLIC
	-Wl,--start-group
	${LIBWEBSOCKETS}
	-Wl,--end-group
	)