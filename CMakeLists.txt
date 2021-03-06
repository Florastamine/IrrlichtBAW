cmake_minimum_required(VERSION 3.1)
project(Irrlicht LANGUAGES CXX C)

# Configure CCache if available
find_program(CCACHE_FOUND ccache)
if(CCACHE_FOUND)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
endif(CCACHE_FOUND)

if((MSVC_VERSION AND NOT ${CMAKE_CXX_COMPILER_VERSION} VERSION_LESS 19.11) OR
  (CMAKE_COMPILER_IS_GNUCXX AND NOT ${CMAKE_CXX_COMPILER_VERSION} VERSION_LESS 7.0) OR
  (CMAKE_CXX_COMPILER_ID STREQUAL "Clang" AND NOT ${CMAKE_CXX_COMPILER_VERSION} VERSION_LESS 3.9))
    set(CMAKE_CXX_STANDARD 17)
else()
    set(CMAKE_CXX_STANDARD 14)
endif()
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF) #...without compiler extensions like gnu++11, but is it really needed?

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/cmake/")

set(CMAKE_CONFIGURATION_TYPES "Debug;Release")
set(LLVM_USE_CRT_DEBUG MTd CACHE STRING "IrrBAW developer note: No idea what subfolder provides this")
set(LLVM_USE_CRT_RELEASE MT CACHE STRING "IrrBAW developer note: No idea what subfolder provides this")

if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type: Debug or Release" FORCE)
endif()

macro(irr_adjust_flags)
	option(IRR_GCC_SANITIZE_ADDRESS OFF)
	option(IRR_GCC_SANITIZE_THREAD OFF)

	if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
		add_compile_options(
			-Wextra
			-Wno-unused-parameter
			-Wno-unused-but-set-parameter
			-fno-exceptions
			-fno-strict-aliasing
			-msse4.2
			-mfpmath=sse
			-fuse-ld=gold
			
			-Wextra
			-Wno-sequence-point
			-Wno-error=ignored-attributes
			-Wno-error=unused-function
			-Wno-error=unused-variable
			-Wno-error=unused-parameter
			
			#"$<$<CONFIG:DEBUG>:-g>" # same as with -O3 below
			"$<$<CONFIG:DEBUG>:-Wall>"
			"$<$<CONFIG:DEBUG>:-fno-omit-frame-pointer>"
			"$<$<CONFIG:DEBUG>:-fstack-protector-strong>"
			#"$<$<CONFIG:RELEASE>:-O3>" # added by default and apparently cmake is not clever enough to filter duplicates for this one
			"$<$<CONFIG:RELEASE>:-fexpensive-optimizations>"
		)
		if (IRR_GCC_SANITIZE_ADDRESS)
			add_compile_options(-fsanitize=address)
		endif()
		if (IRR_GCC_SANITIZE_THREAD)
			add_compile_options(-fsanitize=thread)
		endif()
		if (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 6.1)
			add_compile_options(-Wno-error=ignored-attributes)
		endif()
		
		link_libraries(
			-fuse-ld=gold
		)
		if(FAST_MATH)
			add_compile_options(
				"$<$<CONFIG:RELEASE>:-ffast-math>"
			)
		endif()
		if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
			add_compile_options("$<$CONFIG:DEBUG>:-ggdb3>")
		endif()
	elseif(MSVC) # /arch:sse3 or anything like this is not needed on x64 on MSVC for enabling sse3 instructions
		# debug
		string(REPLACE "/MDd" "/MTd" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
		string(REPLACE "/W3" "/W0" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
		set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /Zc:__cplusplus /MP /fp:fast")
		string(REPLACE "/MDd" "/MTd" CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG}")
		string(REPLACE "/W3" "/W0" CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG}")
		set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /MP /fp:fast")
		
		#release
		string(REPLACE "/GS" "" CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}") # for some reason simply replacing /GS -> /GS- doesn't work... so it vanishes here and appears a few lines below!
		string(REPLACE "/O2" "/Ox" CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
		string(REPLACE "/MD" "/MT" CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
		set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /GL /Zc:__cplusplus /MP /Gy- /Zc:wchar_t /Zi /sdl- /GF /Ot /GS- /Oy /Oi /fp:fast")
		string(REPLACE "/GS" "" CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
		string(REPLACE "/O2" "/Ox" CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
		string(REPLACE "/MD" "/MT" CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
		set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /GL /MP /Gy- /Zc:wchar_t /Zi /sdl- /GF /Ot /GS- /Oy /Oi /fp:fast")
	else()
		message(WARNING "UNTESTED COMPILER DETECTED, EXPECT WRONG OPTIMIZATION FLAGS! SUBMIT ISSUE ON GITHUB https://github.com/buildaworldnet/IrrlichtBAW/issues")
	endif()
endmacro()

macro(irr_adjust_definitions)
	add_definitions(
		-DPNG_THREAD_UNSAFE_OK
		-DPNG_NO_MMX_CODE
		-DPNG_NO_MNG_FEATURES
		-D_7ZIP_ST
		-DSPIRV_CROSS_EXCEPTIONS_TO_ASSERTIONS
	)

	if(WIN32)
		add_definitions(
			-DWIN32
			-D__GNUWIN32__
			-D_CRT_SECURE_NO_DEPRECATE
			-DNOMINMAX
		)
	endif()
endmacro()

if(WIN32)
	set(_IRR_PLATFORM_WINDOWS_ 1)
elseif(UNIX)
	set(_IRR_PLATFORM_LINUX_ 1)
endif()

#uncomment in the future
#option(IRR_TARGET_ARCH_ARM "Build for ARM?" OFF)

option(IRR_COMPILE_WITH_OPENGL "Compile with OpenGL backend?" ON)
option(IRR_COMPILE_WITH_BURNINGSVIDEO "Compile with software backend?" ON)

option(IRR_PCH "Enable pre-compiled header" ON)

option(IRR_FAST_MATH "Enable fast low-precision math" ON)

option(IRR_BUILD_EXAMPLES "Enable building examples" ON)

option(IRR_BUILD_TOOLS "Enable building tools (just convert2BAW as for now)" ON)

set(THIRD_PARTY_SOURCE_DIR "${PROJECT_SOURCE_DIR}/3rdparty")

add_subdirectory(${THIRD_PARTY_SOURCE_DIR})

add_subdirectory(source/Irrlicht) # Irrlicht code

if(IRR_BUILD_TOOLS)
	add_subdirectory(tools) # convert2BAW
endif()
if(IRR_BUILD_EXAMPLES)
	add_subdirectory(examples_tests)
endif()
