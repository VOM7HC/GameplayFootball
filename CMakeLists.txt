cmake_minimum_required(VERSION 3.16)

set(PACKAGE GameplayFootball)

set(VERSION_MAJOR 0)
set(VERSION_MINOR 3)
set(VERSION_PATCH 0)
if(${VERSION_PATCH})
   set(VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH})
else(${VERSION_PATCH})
   set(VERSION ${VERSION_MAJOR}.${VERSION_MINOR})
endif(${VERSION_PATCH})

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/CMakeModules")

project(GameplayFootball)

set (CMAKE_CXX_STANDARD 14)

if(UNIX)
   set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g -Wall")
   # Temporarily suppress warnings during debug
   set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -Wno-unused-variable -Wno-sign-compare -Wno-unused-function -Wno-strict-aliasing")
   set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Wall -O3")
endif(UNIX)
# SET(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} ${GCC_COVERAGE_COMPILE_FLAGS} -fPIC -g -O3")
# SET(CMAKE_C_FLAGS  "${CMAKE_C_FLAGS} -fPIC -O3 -g")

# Use package targets so static dependency libraries and system frameworks
# are carried through to the executable by vcpkg.
find_package(OpenGL REQUIRED)
find_package(SDL2 CONFIG REQUIRED)
find_package(SDL2_image CONFIG REQUIRED)
find_package(SDL2_ttf CONFIG REQUIRED)
find_package(sdl2-gfx CONFIG REQUIRED)
find_package(Boost CONFIG REQUIRED COMPONENTS thread filesystem)
find_package(OpenAL CONFIG REQUIRED)
find_package(SQLite3 REQUIRED)

add_library(gameplay_dependencies INTERFACE)
target_link_libraries(gameplay_dependencies INTERFACE
    OpenGL::GL SDL2::SDL2
    $<IF:$<TARGET_EXISTS:SDL2_image::SDL2_image>,SDL2_image::SDL2_image,SDL2_image::SDL2_image-static>
    $<IF:$<TARGET_EXISTS:SDL2_ttf::SDL2_ttf>,SDL2_ttf::SDL2_ttf,SDL2_ttf::SDL2_ttf-static>
    SDL2::SDL2_gfx Boost::filesystem Boost::thread
    OpenAL::OpenAL SQLite::SQLite3 ${CMAKE_DL_LIBS})
# Object libraries also need the dependency headers while compiling.
link_libraries(gameplay_dependencies)

include_directories(${PROJECT_SOURCE_DIR}/src)

# Include the sources
include(sources.cmake)

# Compile it as multiple static libraries (note: not compiling physics,
# as not used by gameplayfootball)

add_library(baselib OBJECT ${BASE_SOURCES} ${BASE_HEADERS}
        ${BASE_GEOMETRY_HEADERS} ${BASE_MATH_HEADERS})
add_library(systemscommonlib OBJECT ${SYSTEMS_COMMON_SOURCES}
        ${SYSTEMS_COMMON_HEADERS})
add_library(systemsgraphicslib OBJECT ${SYSTEMS_GRAPHICS_SOURCES}
        ${SYSTEMS_GRAPHICS_HEADERS} ${SYSTEMS_GRAPHICS_OBJECTS_HEADERS}
        ${SYSTEMS_GRAPHICS_RESOURCES_HEADERS} ${SYSTEMS_GRAPHICS_RENDERING_HEADERS})
add_library(systemsaudiolib OBJECT ${SYSTEMS_AUDIO_SOURCES}
        ${SYSTEMS_AUDIO_HEADERS} ${SYSTEMS_AUDIO_OBJECTS_HEADERS}
        ${SYSTEMS_AUDIO_RESOURCES_HEADERS} ${SYSTEMS_AUDIO_RENDERING_HEADERS})
add_library(loaderslib OBJECT ${LOADERS_SOURCES} ${LOADERS_HEADERS})
add_library(typeslib OBJECT ${TYPES_SOURCES} ${TYPES_HEADERS})
add_library(frameworklib OBJECT ${FRAMEWORK_SOURCES} ${FRAMEWORK_HEADERS})
add_library(scenelib OBJECT ${SCENE_SOURCES} ${SCENE_HEADERS}
        ${SCENE2D_HEADERS} ${SCENE3D_HEADERS} ${SCENE_OBJECTS_HEADERS}
        ${SCENE_RESOURCES_HEADERS})
add_library(managerslib OBJECT ${MANAGERS_SOURCES} ${MANAGERS_HEADERS})
add_library(utilslib OBJECT ${UTILS_SOURCES} ${UTILS_HEADERS}
        ${UTILS_EXT_HEADERS})
add_library(gui2lib OBJECT ${UTILS_GUI2_SOURCES} ${UTILS_GUI2_HEADERS}
        ${UTILS_GUI2_WIDGETS_HEADERS})

set(OWN_LIBRARIES $<TARGET_OBJECTS:baselib> $<TARGET_OBJECTS:systemscommonlib>
        $<TARGET_OBJECTS:systemsgraphicslib>
        $<TARGET_OBJECTS:systemsaudiolib> $<TARGET_OBJECTS:loaderslib>
        $<TARGET_OBJECTS:typeslib> $<TARGET_OBJECTS:frameworklib>
        $<TARGET_OBJECTS:scenelib> $<TARGET_OBJECTS:managerslib>
        $<TARGET_OBJECTS:utilslib> $<TARGET_OBJECTS:gui2lib>)

# Join all created static libraries into the single static one.
add_library(blunted2 ${BLUNTED_CORE_SOURCES} ${BLUNTED_CORE_HEADERS} ${OWN_LIBRARIES})

add_library(leaguelib ${LEAGUE_SOURCES} ${LEAGUE_HEADERS})
add_library(gamelib ${GAME_SOURCES} ${GAME_HEADERS})
add_library(hidlib ${HID_SOURCES} ${HID_HEADERS})
add_library(menulib ${MENU_SOURCES} ${MENU_HEADERS})
add_library(datalib ${DATA_SOURCES} ${DATA_HEADERS})

set(LIBRARIES gamelib hidlib menulib datalib leaguelib blunted2)

add_executable(gameplayfootball WIN32 ${CORE_SOURCES} ${CORE_HEADERS})
target_link_libraries(gameplayfootball ${LIBRARIES})
