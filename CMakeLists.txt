cmake_minimum_required(VERSION 3.0.2)
project(BabelTrader)

# set compile parameter
if (${CMAKE_CXX_COMPILER_ID} STREQUAL GNU)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native -Wall -Wextra -std=c++11")
elseif (${CMAKE_CXX_COMPILER_ID} MATCHES Clang)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native -Wall -Wextra -Wno-missing-field-initializers")
elseif (${CMAKE_CXX_COMPILER_ID} STREQUAL MSVC)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS=1 -D_UNICODE -DUNICODE)
	# add_definitions(-DNOMINMAX) # for fix std::max conflict in windows
endif()

# set output dir
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# for vim plugin - YCM
if (NOT ${CMAKE_CXX_COMPILER_ID} STREQUAL MSVC)
	set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
endif()

# set use folder in vs
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

# add custom cmake modules
INCLUDE(FindPackageHandleStandardArgs)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_LIST_DIR}/cmake/Modules)
include(${CMAKE_CURRENT_LIST_DIR}/thirdparty/muggleCC/cmake/muggle_utils.cmake)

# dependencies
find_package(Threads)
find_package(OpenSSL REQUIRED)
find_package(ZLIB REQUIRED)
# use libuv and uWebSockets with source code
# find_package(LibUV REQUIRED)
# find_package(uWS REQUIRED)

# libuv
add_subdirectory(${CMAKE_CURRENT_LIST_DIR}/thirdparty/libuv)
set_target_properties(uv uv_a
	PROPERTIES FOLDER "thirdparty/libuv"
)

# uWebSockets
file(GLOB UWS_H ${CMAKE_CURRENT_LIST_DIR}/thirdparty/uWebSockets/src/*.h)
file(GLOB UWS_CPP ${CMAKE_CURRENT_LIST_DIR}/thirdparty/uWebSockets/src/*.cpp)
if (WIN32)
	add_library(uWS SHARED
		${UWS_H}
		${UWS_CPP}
	)
else()
	add_library(uWS SHARED
		${UWS_CPP}
	)
endif(WIN32)
add_dependencies(uWS uv)
target_link_libraries(uWS
	${OPENSSL_LIBRARIES}
	${ZLIB_LIBRARIES}
	${CMAKE_THREAD_LIBS_INIT}
	uv
)
target_include_directories(uWS PUBLIC
	${OPENSSL_INCLUDE_DIR}
	${ZLIB_INCLUDE_DIR}
	${CMAKE_CURRENT_LIST_DIR}/thirdparty/libuv/include
)
if (NOT WIN32)
	target_compile_definitions(uWS PRIVATE UWS_THREADSAFE)
endif()
set_target_properties(uWS
	PROPERTIES FOLDER "thirdparty/uWs"
)
add_custom_command(
	TARGET uWS POST_BUILD
	COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/thirdparty/uWS
)
foreach(uWS_Header ${UWS_H})
	get_filename_component(uWS_Header_name ${uWS_Header} NAME)
	add_custom_command(
		TARGET uWS
		POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E copy ${uWS_Header} ${CMAKE_BINARY_DIR}/thirdparty/uWS/include/uWS/${uWS_Header_name}
	)
endforeach(uWS_Header ${UWS_H})

set_target_properties(uWS PROPERTIES PUBLIC_HEADER "${UWS_H}")
install(
    TARGETS uWS
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib/static
    PUBLIC_HEADER DESTINATION include/uWS
)

# glog
set(WITH_GFLAGS OFF CACHE BOOL "glog with gflags") 
set(BUILD_TESTING OFF CACHE BOOL "glog build testting") 
add_subdirectory(${CMAKE_CURRENT_LIST_DIR}/thirdparty/glog)
set_target_properties(glog
	PROPERTIES FOLDER "thirdparty/google"
)

# muggleCC
set(MUGGLE_BUILD_ONLY_C OFF)
set(MUGGLE_BUILD_EXAMPLE OFF)
set(MUGGLE_BUILD_TEST OFF)
set(MUGGLE_BUILD_BENCHMARK OFF)
add_subdirectory(${CMAKE_CURRENT_LIST_DIR}/thirdparty/muggleCC)
set_target_properties(muggleCC
	PROPERTIES
	FOLDER "thirdparty/muggle"
)

# ctp
muggle_install_headers(${CMAKE_CURRENT_LIST_DIR}/thirdparty/ctp/include include/ctp)
if (WIN32)
	file(GLOB ctp_libs ${CMAKE_CURRENT_LIST_DIR}/thirdparty/ctp/lib/win64/*)
else()
	file(GLOB ctp_libs ${CMAKE_CURRENT_LIST_DIR}/thirdparty/ctp/lib/linux/*)
endif()
install(FILES ${ctp_libs} DESTINATION lib)

# xtp
muggle_install_headers(${CMAKE_CURRENT_LIST_DIR}/thirdparty/xtp/include include/xtp)
if (WIN32)
	file(GLOB xtp_libs ${CMAKE_CURRENT_LIST_DIR}/thirdparty/xtp/lib/win64/*)
else()
	file(GLOB xtp_libs ${CMAKE_CURRENT_LIST_DIR}/thirdparty/xtp/lib/linux/*)
endif()
install(FILES ${xtp_libs} DESTINATION lib)

# dependencies
set(dependencies_inc
	${OPENSSL_INCLUDE_DIR}
	${ZLIB_INCLUDE_DIR}
	${CMAKE_CURRENT_LIST_DIR}/thirdparty/libuv/include
	${CMAKE_BINARY_DIR}/thirdparty/uWS/include/
	${CMAKE_CURRENT_LIST_DIR}/thirdparty/rapidjson/include
	${CMAKE_CURRENT_LIST_DIR}/thirdparty/muggleCC
	${CMAKE_CURRENT_LIST_DIR}/thirdparty/glog/src
	${CMAKE_BINARY_DIR}/thirdparty/glog/
	${CMAKE_CURRENT_LIST_DIR}/src
)

# common
file(GLOB common_h ${CMAKE_CURRENT_LIST_DIR}/src/common/*.h)
file(GLOB common_cpp ${CMAKE_CURRENT_LIST_DIR}/src/common/*.cpp)
if (WIN32)
	add_library(babeltrader-common-cpp STATIC
		${common_h}
		${common_cpp}
	)
else()
	add_library(babeltrader-common-cpp STATIC
		${common_cpp}
	)
endif()
target_include_directories(babeltrader-common-cpp PRIVATE ${dependencies_inc})
target_link_libraries(babeltrader-common-cpp uWS glog)


# utils function
function(add_serv name folder)
	file(GLOB tmp_h ${folder}/*.h)
	file(GLOB tmp_cpp ${folder}/*.cpp)
	if (WIN32)
		add_executable(${name} ${tmp_h} ${tmp_cpp})
	else()
		add_executable(${name} ${tmp_cpp})
	endif(WIN32)
	add_dependencies(${name} uWS glog)
	target_include_directories(${name} PRIVATE ${dependencies_inc})
	target_link_libraries(${name} uWS glog babeltrader-common-cpp)
endfunction(add_serv)

# include and link directories
if (WIN32)
	link_directories(
		${CMAKE_CURRENT_LIST_DIR}/thirdparty/ctp/lib/win64
		${CMAKE_CURRENT_LIST_DIR}/thirdparty/xtp/lib/win64
	)
else()
	link_directories(
		/usr/local/lib
		${CMAKE_CURRENT_LIST_DIR}/thirdparty/ctp/lib/linux
		${CMAKE_CURRENT_LIST_DIR}/thirdparty/xtp/lib/linux
	)
endif()

# performance test
option(BABELTRADER_ENABLE_PERFORMANCE "enable performance" OFF)
if (${BABELTRADER_ENABLE_PERFORMANCE})
	add_definitions(-DENABLE_PERFORMANCE_TEST=1)
else()
	add_definitions(-DENABLE_PERFORMANCE_TEST=0)
endif()

# babeltrader-ctp-quote
add_serv(babeltrader-ctp-quote ${CMAKE_CURRENT_LIST_DIR}/src/ctp-quotes)
target_include_directories(babeltrader-ctp-quote PRIVATE ${CMAKE_CURRENT_LIST_DIR}/thirdparty/ctp/include)
target_link_libraries(babeltrader-ctp-quote thostmduserapi_se)
if (WIN32)
	add_custom_command(
		TARGET babeltrader-ctp-quote
		PRE_LINK
		COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_LIST_DIR}/thirdparty/ctp/lib/win64/thostmduserapi_se.dll $(OutDir)
	)
	set_target_properties(babeltrader-ctp-quote PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY "$(OutDir)")
else()
	add_custom_command(
		TARGET babeltrader-ctp-quote
		PRE_LINK
		COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_LIST_DIR}/thirdparty/ctp/lib/linux/libthostmduserapi_se.so ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/
	)
endif()

# babeltrader-ctp-trade
add_serv(babeltrader-ctp-trade ${CMAKE_CURRENT_LIST_DIR}/src/ctp-trade)
target_include_directories(babeltrader-ctp-trade PRIVATE ${CMAKE_CURRENT_LIST_DIR}/thirdparty/ctp/include)
target_link_libraries(babeltrader-ctp-trade thosttraderapi_se)
if (WIN32)
	add_custom_command(
		TARGET babeltrader-ctp-trade
		PRE_LINK
		COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_LIST_DIR}/thirdparty/ctp/lib/win64/thosttraderapi_se.dll $(OutDir)
	)
	set_target_properties(babeltrader-ctp-trade PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY "$(OutDir)")
else()
	add_custom_command(
		TARGET babeltrader-ctp-trade
		PRE_LINK
		COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_LIST_DIR}/thirdparty/ctp/lib/linux/libthosttraderapi_se.so ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/
	)
endif()

# babeltrader-xtp-trade
add_serv(babeltrader-xtp-trade ${CMAKE_CURRENT_LIST_DIR}/src/xtp-trade)
target_include_directories(babeltrader-xtp-trade PRIVATE ${CMAKE_CURRENT_LIST_DIR}/thirdparty/xtp/include)
target_link_libraries(babeltrader-xtp-trade xtptraderapi)
if (WIN32)
	add_custom_command(
		TARGET babeltrader-xtp-trade
		PRE_LINK
		COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_LIST_DIR}/thirdparty/xtp/lib/win64/xtptraderapi.dll $(OutDir)
	)
	set_target_properties(babeltrader-xtp-trade PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY "$(OutDir)")
	target_compile_definitions(babeltrader-xtp-trade PRIVATE ISLIB)
else()
	add_custom_command(
		TARGET babeltrader-xtp-trade
		PRE_LINK
		COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_LIST_DIR}/thirdparty/xtp/lib/linux/libxtptraderapi.so  ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/
	)
endif()

# babeltrader-xtp-quote
add_serv(babeltrader-xtp-quote ${CMAKE_CURRENT_LIST_DIR}/src/xtp-quotes)
target_include_directories(babeltrader-xtp-quote PRIVATE ${CMAKE_CURRENT_LIST_DIR}/thirdparty/xtp/include)
target_link_libraries(babeltrader-xtp-quote xtpquoteapi)
if (WIN32)
	add_custom_command(
		TARGET babeltrader-xtp-quote
		PRE_LINK
		COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_LIST_DIR}/thirdparty/xtp/lib/win64/xtpquoteapi.dll $(OutDir)
	)
	set_target_properties(babeltrader-xtp-quote PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY "$(OutDir)")
	target_compile_definitions(babeltrader-xtp-quote PRIVATE ISLIB)
else()
	add_custom_command(
		TARGET babeltrader-xtp-quote
		PRE_LINK
		COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_LIST_DIR}/thirdparty/xtp/lib/linux/libxtpquoteapi.so ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/
	)
endif()

# fake target
if (WIN32)
	add_custom_target(
		FAKE_COPY_CONFIG ALL
		COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_LIST_DIR}/config ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/$(Configuration)/config
	)
else()
	add_custom_target(
		FAKE_COPY_CONFIG ALL
		COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_LIST_DIR}/config ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/config
	)
endif()

# cpp demo
add_serv(test_quote ${CMAKE_CURRENT_LIST_DIR}/demo/cpp/test_quote)

if (WIN32)
	set_target_properties(test_quote
		PROPERTIES
		FOLDER "demo"
		VS_DEBUGGER_WORKING_DIRECTORY "$(OutDir)"
	)
endif()
