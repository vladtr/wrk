cmake_minimum_required(VERSION 3.10)
project(wrk LANGUAGES C)

# Set the C standard to C99
set(CMAKE_C_STANDARD 99)

# Add general compiler flags
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -O2 -D_REENTRANT")

# Detect the platform and set specific flags
if(${CMAKE_SYSTEM_NAME} MATCHES "SunOS")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_PTHREADS -D_POSIX_C_SOURCE=200112L")
    set(LIBS ${LIBS} socket)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(CMAKE_OSX_DEPLOYMENT_TARGET "${CMAKE_OSX_DEPLOYMENT_TARGET}")
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_POSIX_C_SOURCE=200112L -D_BSD_SOURCE -D_DEFAULT_SOURCE")
    set(LIBS ${LIBS} dl)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-E")
elseif(${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_DECLARE_C99_LDBL_MATH")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-E")
endif()

# Find OpenSSL
find_package(OpenSSL REQUIRED)
include_directories(${OPENSSL_INCLUDE_DIR})
list(APPEND LIBS ${OPENSSL_LIBRARIES})

# LuaJIT configuration
# Automatically detect the correct include path for LuaJIT
find_path(LUAJIT_INCLUDE_DIR
    NAMES lua.h
    PATHS /usr/local/include/luajit-2.1 /usr/local/include/luajit-2.0 /usr/include/luajit-2.1 /usr/include/luajit-2.0
)

if(NOT LUAJIT_INCLUDE_DIR)
    message(FATAL_ERROR "LuaJIT not found. Please install LuaJIT or set the correct path.")
endif()

# Set library path for LuaJIT
find_library(LUAJIT_LIB
    NAMES luajit-5.1
    PATHS /usr/local/lib /usr/lib
)

if(NOT LUAJIT_LIB)
    message(FATAL_ERROR "LuaJIT library not found. Please install LuaJIT or set the correct library path.")
endif()

include_directories(${LUAJIT_INCLUDE_DIR})
link_directories(${LUAJIT_LIB})

list(APPEND LIBS ${LUAJIT_LIB})

# Define the source directory explicitly
set(SOURCE_DIR ${CMAKE_SOURCE_DIR}/src)

# Source files
set(SRC_FILES
    ${SOURCE_DIR}/wrk.c
    ${SOURCE_DIR}/net.c
    ${SOURCE_DIR}/ssl.c
    ${SOURCE_DIR}/aprintf.c
    ${SOURCE_DIR}/stats.c
    ${SOURCE_DIR}/script.c
    ${SOURCE_DIR}/units.c
    ${SOURCE_DIR}/ae.c
    ${SOURCE_DIR}/zmalloc.c
    ${SOURCE_DIR}/http_parser.c
)

# Define the executable
add_executable(wrk ${SRC_FILES})

# Link libraries
target_link_libraries(wrk ${LUAJIT_LIB} ${LIBS} m pthread)

# Generate version.c file containing version string
set(VERSION_STR "1.0")  # You can change this version as needed
file(WRITE ${CMAKE_BINARY_DIR}/version.c "const char *VERSION = \"${VERSION_STR}\";\n")

# Compile version.c into version.o
add_library(version_obj OBJECT ${CMAKE_BINARY_DIR}/version.c)

# Link the custom object files (version_obj) with the main executable
target_sources(wrk PRIVATE $<TARGET_OBJECTS:version_obj>)
