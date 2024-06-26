cmake_minimum_required(VERSION 3.18.4)

option(RSDKv5_WIIU_SKIP_PKGCONFIG "Wii U pkg-config doesn't work right now on Windows, so use this to skip it until it's fixed." ON)
option(RSDKv5_WIIU_EXPERIMENTAL_SDL2 "Use GaryOderNichts' custom updated SDL2 Wii U port." ON)
option(RSDKv5_STATIC "Static-link the dependencies." OFF)
option(RSDKv5_USE_LIBVORBIS "Use libvorbis instead of stb_vorbis." OFF)

project(RSDKv5 LANGUAGES C CXX)

add_library(RSDKv5 OBJECT
	"../RSDKv5/main.cpp"
	"../RSDKv5/RSDK/Core/RetroEngine.cpp"
	"../RSDKv5/RSDK/Core/Math.cpp"
	"../RSDKv5/RSDK/Core/Reader.cpp"
	"../RSDKv5/RSDK/Core/Link.cpp"
	"../RSDKv5/RSDK/Core/ModAPI.cpp"
	"../RSDKv5/RSDK/Dev/Debug.cpp"
	"../RSDKv5/RSDK/Storage/Storage.cpp"
	"../RSDKv5/RSDK/Storage/Text.cpp"
	"../RSDKv5/RSDK/Graphics/Drawing.cpp"
	"../RSDKv5/RSDK/Graphics/Scene3D.cpp"
	"../RSDKv5/RSDK/Graphics/Animation.cpp"
	"../RSDKv5/RSDK/Graphics/Sprite.cpp"
	"../RSDKv5/RSDK/Graphics/Palette.cpp"
	"../RSDKv5/RSDK/Graphics/Video.cpp"
	"../RSDKv5/RSDK/Audio/Audio.cpp"
	"../RSDKv5/RSDK/Input/Input.cpp"
	"../RSDKv5/RSDK/Scene/Scene.cpp"
	"../RSDKv5/RSDK/Scene/Collision.cpp"
	"../RSDKv5/RSDK/Scene/Object.cpp"
	"../RSDKv5/RSDK/Scene/Objects/DefaultObject.cpp"
	"../RSDKv5/RSDK/Scene/Objects/DevOutput.cpp"
	"../RSDKv5/RSDK/User/Core/UserAchievements.cpp"
	"../RSDKv5/RSDK/User/Core/UserCore.cpp"
	"../RSDKv5/RSDK/User/Core/UserLeaderboards.cpp"
	"../RSDKv5/RSDK/User/Core/UserPresence.cpp"
	"../RSDKv5/RSDK/User/Core/UserStats.cpp"
	"../RSDKv5/RSDK/User/Core/UserStorage.cpp"
	"../dependencies/all/tinyxml2/tinyxml2.cpp"
	"../dependencies/all/iniparser/iniparser.cpp"
	"../dependencies/all/iniparser/dictionary.cpp"
	"../dependencies/all/miniz/miniz.c"
)

target_include_directories(RSDKv5 PRIVATE
	"../RSDKv5"
	"../dependencies/all"
	"../dependencies/all/tinyxml2"
)

target_compile_definitions(RSDKv5 PUBLIC # Public because the game itself relies on RETRO_STANDALONE and RETRO_REVISION.
	RETRO_STANDALONE=0
	RSDK_USE_SDL2
	RETRO_REVISION=2
	RETRO_DISABLE_LOG	# Disable logging because even just destroying an enemy will cause a debug message to be logged which causes the Wii U build to stutter.
)

if(RSDKv5_STATIC)
	target_link_options(RSDKv5 PUBLIC "-static")
endif()

if(RSDKv5_USE_LIBVORBIS)
	target_compile_definitions(RSDKv5 PRIVATE RSDKv5_USE_LIBVORBIS)
endif()

if(RSDKv5_WIIU_EXPERIMENTAL_SDL2)
	add_subdirectory("../dependencies/wiiu/SDL" EXCLUDE_FROM_ALL "SDL")
	target_link_libraries(RSDKv5 PUBLIC SDL2::SDL2-static SDL2::SDL2main) # Public because SDL2 may use a define to redirect the main function.
endif()

if(RSDKv5_WIIU_SKIP_PKGCONFIG)
	if(NOT RSDKv5_WIIU_EXPERIMENTAL_SDL2)
		target_compile_options(RSDKv5 PRIVATE "-I${DEVKITPRO}/portlibs/wiiu/include/SDL2")
		target_link_libraries(RSDKv5 PRIVATE "-L${DEVKITPRO}/portlibs/wiiu/lib -lSDL2 -lwut -lm -lm -Wl,--no-undefined -lSDL2 -lwut -lm")
	endif()

	target_compile_options(RSDKv5 PRIVATE "-I${DEVKITPRO}/portlibs/ppc/include")
	target_link_libraries(RSDKv5 PRIVATE "-L${DEVKITPRO}/portlibs/ppc/lib -ltheora -logg")

	if(RSDKv5_USE_LIBVORBIS)
		target_compile_options(RSDKv5 PRIVATE "-I${DEVKITPRO}/portlibs/ppc/include")
		target_link_libraries(RSDKv5 PRIVATE "-L${DEVKITPRO}/portlibs/ppc/lib -lvorbisfile -lvorbis -lm -logg")
	endif()
else()
	function(link_library_with_pkgconf target visibility lib)
		pkg_check_modules(${lib} REQUIRED IMPORTED_TARGET ${lib})

		if(RSDKv5_STATIC)
			target_compile_options(${target} ${VISIBILITY} ${${lib}_STATIC_CFLAGS})
			target_link_libraries(${target} PRIVATE ${${lib}_STATIC_LDFLAGS})
		else()
			target_compile_options(${target} ${visibility} ${${lib}_CFLAGS})
			target_link_libraries(${target} PRIVATE ${${lib}_LDFLAGS})
		endif()
	endfunction()

	find_package(PkgConfig REQUIRED)

	if(NOT RSDKv5_WIIU_EXPERIMENTAL_SDL2)
		if(MINGW)
			# SDL2main and libmingw32 have a circular dependency issue
			# that breaks MinGW right now, so fix it by doing this.
			target_link_libraries(RSDKv5 PUBLIC mingw32)
		endif()

		link_library_with_pkgconf(RSDKv5 PUBLIC sdl2)
	endif()

	link_library_with_pkgconf(RSDKv5 PRIVATE theoradec)

	if(RSDKv5_USE_LIBVORBIS)
		link_library_with_pkgconf(RSDKv5 PRIVATE vorbisfile)
	endif()
endif()

if(NOT WIIU)
	target_link_libraries(RSDKv5 PRIVATE ${CMAKE_DL_LIBS})
endif()
