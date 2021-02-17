# Set the minimum version of CMake that can be used
# To find the cmake version run
# $ cmake --version
cmake_minimum_required(VERSION 3.5)

# Set the project name
project (LandAndWaves)

add_definitions(-DUNICODE -D_UNICODE)
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /SUBSYSTEM:WINDOWS")

# Create a sources variable with a link to all cpp files to compile
set(SOURCES
	src/d3dApp.cpp
	src/d3dUtil.cpp
	src/GameTimer.cpp
	src/FrameResource.cpp
	src/Waves.cpp
	src/MathHelper.cpp
	src/DDSTextureLoader.cpp
	src/GeometryGenerator.cpp
	src/LandAndWavesApp.cpp
	
	include/d3dApp.h
	include/Waves.h
	include/d3dUtil.h
	include/d3dx12.h
	include/DDSTextureLoader.h
	include/GameTimer.h
	include/MathHelper.h
	include/UploadBuffer.h
	include/FrameResource.h
	include/GeometryGenerator.h
)

# Add an executable with the above sources
add_executable(LandAndWaves ${SOURCES})

# Set the directories that should be included in the build command for this target
# when running g++ these will be included as -I/directory/path/
target_include_directories(LandAndWaves
    PRIVATE 
        ${PROJECT_SOURCE_DIR}/include
)

#target_link_libraries(LandAndWaves ws2_32.lib)