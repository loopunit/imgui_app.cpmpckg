cmake_minimum_required( VERSION 3.12 )

project(imgui_app
	VERSION 1.7.8)

include(CMakeDependentOption)

option(IMGUI_BUILD_EXAMPLES "Build examples." OFF)
cmake_dependent_option(IMGUI_BUILD_APP_FW "Build app framework." OFF "IMGUI_BUILD_EXAMPLES" ON)

# ---- Add dependencies via CPM ----
# see https://github.com/TheLartians/CPM.cmake for more info

include(ExternalProject)

####

if (NOT DEFINED CPM_SCRIPTS)
	include(cmake/CPM.cmake)

	# One frustrating part of this project setup is the circular dependency between cpmpckg and this repo.
	# GIT_TAG will always lag cpmpckg @ HEAD when this project is updated there.
	CPMAddPackage(
	  NAME cpmpckg
	  GITHUB_REPOSITORY loopunit/cpmpckg
	  GIT_TAG 9d5271a66365729b54588d166ec672d7db0a039c
	  DOWNLOAD_ONLY true)

	include(${cpmpckg_SOURCE_DIR}/cmake/add_cpm_module.cmake)
else()
	include(${CPM_SCRIPTS}/CPM.cmake)
	include(${CPM_SCRIPTS}/add_cpm_module.cmake)
endif()

####

CPMAddPackage(
  NAME PackageProject.cmake
  GITHUB_REPOSITORY loopunit/PackageProject.cmake
  GIT_TAG e5ec20069766f4f078f9f01a86e250e20da0817c)

####

CPMAddBaseModule(imgui_app_fw)
CPMAddBaseModule(imgui_addons)

###################

file(GLOB imgui_app_sources 
	${CMAKE_CURRENT_SOURCE_DIR}/src/imgui_app_internal.h
	${CMAKE_CURRENT_SOURCE_DIR}/src/imgui_app_logger.h
	${CMAKE_CURRENT_SOURCE_DIR}/src/imgui_app_logger.cpp
	${CMAKE_CURRENT_SOURCE_DIR}/src/imgui_app.cpp)

###################

CPMAddPackage(
NAME atomic_queue
	GITHUB_REPOSITORY max0x7ba/atomic_queue
	GIT_TAG df794032353a9fbf6204481bdd53ef0263ce51f3 
	DOWNLOAD_ONLY true)

if( NOT ${atomic_queue_ADDED} )
	message( FATAL_ERROR "Could not find atomic_queue" )
endif()

install(
	DIRECTORY ${atomic_queue_SOURCE_DIR}/include/atomic_queue
	DESTINATION include)

list(APPEND imgui_public_include_paths ${atomic_queue_SOURCE_DIR}/include)

#############

CPMAddPackage(
	NAME spdlog
	GITHUB_REPOSITORY gabime/spdlog
	GIT_TAG 18e3f07f7d333d33c0ef8fbea02299dbe5d49f97 
	DOWNLOAD_ONLY true)

if( NOT ${spdlog_ADDED} )
	message( FATAL_ERROR "Could not find spdlog" )
endif()

install(
	DIRECTORY ${spdlog_SOURCE_DIR}/include/spdlog
	DESTINATION include)

file(GLOB spdlog_source 
	${spdlog_SOURCE_DIR}/src/*.cpp)

list(APPEND imgui_public_include_paths ${spdlog_SOURCE_DIR}/include)
list(APPEND imgui_app_sources ${spdlog_source})

#############

CPMAddPackage(
	NAME fmt
	GITHUB_REPOSITORY fmtlib/fmt
	GIT_TAG af283059612dd06c321dc509255ae9b003ca5e69
	DOWNLOAD_ONLY true)

if( NOT ${fmt_ADDED} )
	message( FATAL_ERROR "Could not find fmt" )
endif()

install(
	DIRECTORY ${fmt_SOURCE_DIR}/include/fmt
	DESTINATION include)

file(GLOB fmt_source 
	${fmt_SOURCE_DIR}/src/*.cc)

list(APPEND imgui_public_include_paths ${fmt_SOURCE_DIR}/include)
list(APPEND imgui_app_sources ${fmt_source})

#############

CPMAddPackage(
	NAME tinyfsm
	GITHUB_REPOSITORY digint/tinyfsm
	GIT_TAG 06f27072266d6c34d990f910c4c43fb11240501c
	DOWNLOAD_ONLY true)

if( NOT ${tinyfsm_ADDED} )
	message( FATAL_ERROR "Could not find tinyfsm" )
endif()

install(
	FILES ${tinyfsm_SOURCE_DIR}/include/tinyfsm.hpp
	DESTINATION include)

list(APPEND imgui_public_include_paths ${tinyfsm_SOURCE_DIR}/include)

#############

CPMAddPackage(
	NAME nfd
	GITHUB_REPOSITORY native-toolkit/nfd
	GIT_TAG e9e3636b031797396aa502b78432ae1347c2afe3
	DOWNLOAD_ONLY true)

if( NOT ${nfd_ADDED} )
	message( FATAL_ERROR "Could not find nfd" )
endif()

install(
	FILES ${nfd_SOURCE_DIR}/src/include/nfd.h
	DESTINATION include)

file(GLOB nfd_source 
	${nfd_SOURCE_DIR}/src/nfd_win.cpp
	${nfd_SOURCE_DIR}/src/common.h
	${nfd_SOURCE_DIR}/src/nfd_common.h
	${nfd_SOURCE_DIR}/src/nfd_common.c)

list(APPEND imgui_public_include_paths ${nfd_SOURCE_DIR}/src)
list(APPEND imgui_public_include_paths ${nfd_SOURCE_DIR}/src/include)

list(APPEND imgui_app_sources ${nfd_source})

#############

CPMAddPackage(
	NAME boxer
	GITHUB_REPOSITORY aaronmjacobs/Boxer
	GIT_TAG 01c17f6a9cd66068f7890ea887ab3b9a673f0434
	DOWNLOAD_ONLY true)

if( NOT ${boxer_ADDED} )
	message( FATAL_ERROR "Could not find boxer" )
endif()

install(
	DIRECTORY ${boxer_SOURCE_DIR}/include/boxer
	DESTINATION include)

if(APPLE)
	file(GLOB boxer_source 
		${boxer_SOURCE_DIR}/src/boxer_osx.mm)
elseif (WIN32)
	file(GLOB boxer_source 
		${boxer_SOURCE_DIR}/src/boxer_win.cpp)
elseif (LINUX)
	file(GLOB boxer_source 
		${boxer_SOURCE_DIR}/src/boxer_linux.cpp)
endif()

list(APPEND imgui_public_include_paths ${boxer_SOURCE_DIR}/include)

list(APPEND imgui_app_sources ${boxer_source})

#############

list(APPEND imgui_private_include_paths ${PROJECT_BINARY_DIR}/include)

add_library(imgui_app STATIC ${imgui_app_sources})

target_link_libraries(imgui_app PUBLIC cpm_runtime::imgui_addons cpm_runtime::imgui_app_fw)

target_include_directories(imgui_app PRIVATE ${imgui_private_include_paths} ${CMAKE_CURRENT_LIST_DIR}/src)

target_include_directories(imgui_app 
	PUBLIC 
		${imgui_public_include_paths} 
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> 
		$<INSTALL_INTERFACE:include>)

set_target_properties(imgui_app PROPERTIES CXX_STANDARD 17)

target_compile_definitions(imgui_app PUBLIC SPDLOG_COMPILED_LIB SPDLOG_FMT_EXTERNAL)

packageProject(
	# the name of the target to export
	NAME imgui_app
	# the version of the target to export
	VERSION ${PROJECT_VERSION}
	# a temporary directory to create the config files
	BINARY_DIR ${PROJECT_BINARY_DIR}
	# location of the target's public headers
	INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include
	# should match the target's INSTALL_INTERFACE include directory
	INCLUDE_DESTINATION include
	# semicolon separated list of the project's dependencies
	DEPENDENCIES "")
  
if(IMGUI_BUILD_EXAMPLES)
	file(GLOB example_sources 
		${CMAKE_CURRENT_SOURCE_DIR}/examples/app_main.cpp)

	add_executable(imgui_app_example
		${example_sources})
    
	set_target_properties(imgui_app_example PROPERTIES CXX_STANDARD 17)

	target_include_directories(imgui_app_example
		PRIVATE 
			${imgui_SOURCE_DIR}/examples)
    
	target_link_libraries(imgui_app_example
		PUBLIC
			imgui_app)
endif()