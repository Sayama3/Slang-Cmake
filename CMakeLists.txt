cmake_minimum_required(VERSION 3.14)

project(SlangCmake)

option(SLANG_FORCE_FETCH "" OFF)

option(SLANG_GIT_REPOSITORY "Repository where we search for the shader slang library." https://github.com/shader-slang/slang.git)
if(NOT SLANG_GIT_REPOSITORY)
	set(SLANG_GIT_REPOSITORY "https://github.com/shader-slang/slang.git")
endif()

option(SLANG_GIT_TAG "Version of the slang library we are fetching." v2025.19.1)
if(NOT SLANG_GIT_TAG)
	set(SLANG_GIT_TAG "v2025.19.1")
endif()

message(STATUS "Vulkan SDK: $ENV{VULKAN_SDK}")
message(STATUS "SLANG_FORCE_FETCH: ${SLANG_FORCE_FETCH}")
if(SLANG_FORCE_FETCH)
	message(STATUS "SLANG_GIT_REPOSITORY: ${SLANG_GIT_REPOSITORY}")
	message(STATUS "SLANG_GIT_TAG: ${SLANG_GIT_TAG}")
endif()

if ((NOT SLANG_FORCE_FETCH) AND (DEFINED ENV{VULKAN_SDK}))

	if (NOT SLANG_ROOT)
		set(SLANG_ROOT $ENV{VULKAN_SDK})
	endif ()

	message(STATUS "Searching for Slang in: ${SLANG_ROOT}")

	if(IOS)
		get_filename_component(Vulkan_Target_SDK "$ENV{VULKAN_SDK}/.." REALPATH)
		list(APPEND CMAKE_FRAMEWORK_PATH "${Vulkan_Target_SDK}/iOS/lib")
		set (CMAKE_FIND_ROOT_PATH_MODE_PROGRAM BOTH)
		set (CMAKE_FIND_ROOT_PATH_MODE_LIBRARY BOTH)
		set (CMAKE_FIND_ROOT_PATH_MODE_INCLUDE BOTH)
	endif()

	if(WIN32)
		set(_Slang_hint_include_search_paths
				"$ENV{VULKAN_SDK}/include"
		)
		if(CMAKE_SIZEOF_VOID_P EQUAL 8)
			set(_Slang_hint_executable_search_paths
					"$ENV{VULKAN_SDK}/bin"
			)
			set(_Slang_hint_library_search_paths
					"$ENV{VULKAN_SDK}/lib"
					"$ENV{VULKAN_SDK}/bin"
			)
		else()
			set(_Slang_hint_executable_search_paths
					"$ENV{VULKAN_SDK}/bin32"
					"$ENV{VULKAN_SDK}/bin"
			)
			set(_Slang_hint_library_search_paths
					"$ENV{VULKAN_SDK}/lib32"
					"$ENV{VULKAN_SDK}/bin32"
					"$ENV{VULKAN_SDK}/lib"
					"$ENV{VULKAN_SDK}/bin"
			)
		endif()
	else()
		set(_Slang_hint_include_search_paths
				"$ENV{VULKAN_SDK}/include"
		)
		set(_Slang_hint_executable_search_paths
				"$ENV{VULKAN_SDK}/bin"
		)
		set(_Slang_hint_library_search_paths
				"$ENV{VULKAN_SDK}/lib"
		)
	endif()
	if(APPLE AND DEFINED ENV{VULKAN_SDK})
		list(APPEND _Slang_hint_include_search_paths
				"${Vulkan_Target_SDK}/macOS/include"
		)
		if(CMAKE_SYSTEM_NAME STREQUAL "iOS")
			list(APPEND _Slang_hint_library_search_paths
					"${Vulkan_Target_SDK}/iOS/lib"
			)
		elseif(CMAKE_SYSTEM_NAME STREQUAL "tvOS")
			list(APPEND _Slang_hint_library_search_paths
					"${Vulkan_Target_SDK}/tvOS/lib"
			)
		else()
			list(APPEND _Slang_hint_library_search_paths
					"${Vulkan_Target_SDK}}/lib"
			)
		endif()
	endif()

	find_path(Slang_INCLUDE_DIR
			NAMES slang/slang.h
			HINTS
			${_Slang_hint_include_search_paths}
	)
	mark_as_advanced(Slang_INCLUDE_DIR)

	find_library(Slang_LIBRARY
			NAMES slang
			HINTS
			${_Slang_hint_library_search_paths})
	mark_as_advanced(Slang_LIBRARY)

	find_library(Slang_DEBUG_LIBRARY
			NAMES slangd
			HINTS
			${_Slang_hint_library_search_paths})
	mark_as_advanced(Slang_DEBUG_LIBRARY)

	if((Slang_LIBRARY OR Slang_DEBUG_LIBRARY) AND NOT TARGET slang)
		add_library(slang STATIC IMPORTED)

		set_property(TARGET slang
				PROPERTY
				INTERFACE_INCLUDE_DIRECTORIES "${Slang_INCLUDE_DIR}")

		if(Slang_LIBRARY)
			set_property(TARGET slang APPEND
					PROPERTY
					IMPORTED_CONFIGURATIONS Release)
			set_property(TARGET slang
					PROPERTY
					IMPORTED_LOCATION_RELEASE "${Slang_LIBRARY}")
			message(STATUS "Found Slang Release Mode.")
		endif()

		if(Slang_DEBUG_LIBRARY)
			set_property(TARGET slang APPEND
					PROPERTY
					IMPORTED_CONFIGURATIONS Debug)
			set_property(TARGET slang
					PROPERTY
					IMPORTED_LOCATION_DEBUG "${Slang_DEBUG_LIBRARY}")
			message(STATUS "Found Slang Debug Mode.")
		endif()

		if(UNIX)
			find_package(Threads REQUIRED)
			target_link_libraries(slang
					INTERFACE
					Threads::Threads)
		endif()
	endif()

endif ()

if (NOT TARGET slang)
	if(NOT SLANG_FORCE_FETCH)
		message(WARNING "Slang Not found, fetch from the repository \"${SLANG_GIT_REPOSITORY}\" at tag \"${SLANG_GIT_TAG}\"")
	endif()

	include(FetchContent)

	# Configure Slang options before making it available
	set(SLANG_EMBED_CORE_MODULE ON)
	set(SLANG_EMBED_CORE_MODULE_SOURCE ON)
	set(SLANG_LIB_TYPE STATIC)
	set(SLANG_ENABLE_TESTS OFF)
	set(SLANG_ENABLE_EXAMPLES OFF)
	set(SLANG_ENABLE_GFX OFF)

	FetchContent_Declare(slang
			GIT_REPOSITORY "${SLANG_GIT_REPOSITORY}"
			GIT_TAG "${SLANG_GIT_TAG}"
			PATCH_COMMAND git apply ${CMAKE_CURRENT_SOURCE_DIR}/slang-imgui.patch
	)
	FetchContent_MakeAvailable(slang)
endif ()

add_library(SlangCmake::slang ALIAS slang)
message(STATUS "Library \"SlangCmake::slang\" created.")