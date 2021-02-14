# Set the minimum version of CMake that can be used
# To find the cmake version run
# $ cmake --version
cmake_minimum_required(VERSION 3.5)

# Set the project name
project (Box7)

add_definitions(-DUNICODE -D_UNICODE)
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /SUBSYSTEM:WINDOWS")

# Create a sources variable with a link to all cpp files to compile
set(SOURCES
	src/d3dApp.cpp
	src/d3dUtil.cpp
	src/GameTimer.cpp
	src/BoxApp.cpp
	src/MathHelper.cpp
	src/DDSTextureLoader.cpp
	
	include/d3dApp.h
	include/d3dUtil.h
	include/d3dx12.h
	include/DDSTextureLoader.h
	include/GameTimer.h
	include/MathHelper.h
	include/UploadBuffer.h
)

# Add an executable with the above sources
add_executable(Box7 ${SOURCES})

# Set the directories that should be included in the build command for this target
# when running g++ these will be included as -I/directory/path/
target_include_directories(Box7
    PRIVATE 
        ${PROJECT_SOURCE_DIR}/include
)

#target_link_libraries(Box7 ws2_32.lib)