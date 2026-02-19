
cmake_minimum_required(VERSION 2.6)

message(STATUS "PROJECT PATH IS: :${PROJECT_SOURCE_DIR}")
message(STATUS "architecture is::${THIRDPARTY_STATIC_BUILD}")

# Default NPROC to 4 if not set
if (NOT NPROC)
    set(NPROC 4)
endif()

if (THIRDPARTY_STATIC_BUILD STREQUAL "mips")
    set(ARCH_FLAG "linux-mips32")
    set(CROSS_C_COMPILER ${CMAKE_C_COMPILER})
    # Extract directory and short name for cross-compile-prefix
    get_filename_component(CROSS_TOOLCHAIN_DIR "${CMAKE_C_COMPILER}" DIRECTORY)
    get_filename_component(CROSS_GCC_NAME "${CMAKE_C_COMPILER}" NAME)
    string(REGEX REPLACE "-gcc$" "-" CROSS_SHORT_PREFIX "${CROSS_GCC_NAME}")
    string(REGEX REPLACE "-gcc$" "-ar" CROSS_AR "${CMAKE_C_COMPILER}")
    string(REGEX REPLACE "-gcc$" "-ranlib" CROSS_RANLIB "${CMAKE_C_COMPILER}")
elseif (THIRDPARTY_STATIC_BUILD STREQUAL "arm")
    set(ARCH_FLAG "linux-armv4")
    set(CROSS_C_COMPILER ${CMAKE_C_COMPILER})
    # Extract directory and short name for cross-compile-prefix
    get_filename_component(CROSS_TOOLCHAIN_DIR "${CMAKE_C_COMPILER}" DIRECTORY)
    get_filename_component(CROSS_GCC_NAME "${CMAKE_C_COMPILER}" NAME)
    string(REGEX REPLACE "-gcc$" "-" CROSS_SHORT_PREFIX "${CROSS_GCC_NAME}")
    string(REGEX REPLACE "-gcc$" "-ar" CROSS_AR "${CMAKE_C_COMPILER}")
    string(REGEX REPLACE "-gcc$" "-ranlib" CROSS_RANLIB "${CMAKE_C_COMPILER}")
endif()



#openssl compile and add libs to libs file
message(STATUS "compiling openssl")
execute_process(
    COMMAND rm -rf ../libs
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/openssl-1.1.1m"
)
execute_process(
    COMMAND mkdir ../libs
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/openssl-1.1.1m"
)

execute_process(
    COMMAND chmod +x config
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/openssl-1.1.1m"
)

execute_process(
    COMMAND chmod +x Configure
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/openssl-1.1.1m"
)

if (THIRDPARTY_STATIC_BUILD STREQUAL "ON")
    execute_process(
        COMMAND ./config --static
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/openssl-1.1.1m"
    )
elseif (THIRDPARTY_STATIC_BUILD STREQUAL "mips" OR THIRDPARTY_STATIC_BUILD STREQUAL "arm")
    # Add toolchain bin dir to PATH so short prefix works
    set(ENV{PATH} "${CROSS_TOOLCHAIN_DIR}:$ENV{PATH}")
    execute_process(
        COMMAND ./Configure ${ARCH_FLAG} --static no-shared no-async --cross-compile-prefix=${CROSS_SHORT_PREFIX}
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/openssl-1.1.1m"
    )
endif(THIRDPARTY_STATIC_BUILD)

execute_process(
    COMMAND make -j${NPROC}
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/openssl-1.1.1m"
)

execute_process(
    COMMAND cp -f libcrypto.a ../libs
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/openssl-1.1.1m"
)

execute_process(
    COMMAND cp -f libssl.a ../libs
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/openssl-1.1.1m"
)

execute_process(
    COMMAND cp -rf include/openssl ../include
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/openssl-1.1.1m"
)

#libevent compile and add libs to libs file
message(STATUS "compiling libevnet")
execute_process(
    COMMAND rm -rf build
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/libevent-2.1.10"
)

execute_process(
    COMMAND mkdir build
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/libevent-2.1.10"
)
if (THIRDPARTY_STATIC_BUILD STREQUAL "ON")
    execute_process(
        COMMAND cmake -DOPENSSL_ROOT_DIR=${PROJECT_SOURCE_DIR}/thirdparty/openssl-1.1.1m -DOPENSSL_LIBRARES=${PROJECT_SOURCE_DIR}/thirdparty/openssl-1.1.1m ..
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/libevent-2.1.10/build"
    )
else()
    execute_process(
        COMMAND cmake -DCMAKE_C_COMPILER=${CROSS_C_COMPILER} -DCMAKE_SYSTEM_NAME=Linux -DOPENSSL_ROOT_DIR=${PROJECT_SOURCE_DIR}/thirdparty/openssl-1.1.1m -DOPENSSL_LIBRARIES=${PROJECT_SOURCE_DIR}/thirdparty/openssl-1.1.1m -DEVENT__DISABLE_TESTS=ON -DEVENT__DISABLE_SAMPLES=ON -DEVENT__DISABLE_BENCHMARK=ON ..
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/libevent-2.1.10/build"
    )
endif(THIRDPARTY_STATIC_BUILD)

execute_process(
    COMMAND make -j${NPROC}
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/libevent-2.1.10/build"
)
# wait libevent create libs
set(EVENT_BUILD_PATH ${PROJECT_SOURCE_DIR}/thirdparty/libevent-2.1.10/build)
set(LIBS_PATH ${PROJECT_SOURCE_DIR}/thirdparty/libs)

file(GLOB EVENT_LIBS_PATH
    "${EVENT_BUILD_PATH}/lib/libevent*.a"	
)
file(COPY ${EVENT_LIBS_PATH} DESTINATION ${LIBS_PATH})

# Copy ARM-specific event-config.h
if (THIRDPARTY_STATIC_BUILD STREQUAL "arm" OR THIRDPARTY_STATIC_BUILD STREQUAL "mips")
    execute_process(
        COMMAND cp -f build/include/event2/event-config.h ../include/libevent/event2/event-config.h
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/libevent-2.1.10"
    )
endif()

#json-c compile and add libs to libs file
message(STATUS "compiling json-c")
execute_process(
    COMMAND rm -rf build
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/json-c"
)

execute_process(
    COMMAND mkdir build
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/json-c"
)
if (THIRDPARTY_STATIC_BUILD STREQUAL "ON")
    execute_process(
        COMMAND cmake ..
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/json-c/build"
    )
else()
    execute_process(
        COMMAND cmake -DCMAKE_C_COMPILER=${CROSS_C_COMPILER} ..
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/json-c/build"
    )
endif(THIRDPARTY_STATIC_BUILD)

execute_process(
    COMMAND make -j${NPROC}
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/json-c/build"
)


execute_process(
    COMMAND cp -f libjson-c.a ../../libs
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/json-c/build"
)

execute_process(
    COMMAND cp -f json.h ../../include/json-c
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/json-c/build"
)

# compile mips or arm
if (THIRDPARTY_STATIC_BUILD STREQUAL "arm" OR THIRDPARTY_STATIC_BUILD STREQUAL "mips")
    execute_process(
        COMMAND make -f Makefile.in clean
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/zlib-1.2.13"
        ERROR_QUIET
    )
    execute_process(
        COMMAND chmod +x configure
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/zlib-1.2.13"
    )
    set(ENV{CC} "${CROSS_C_COMPILER}")
    set(ENV{AR} "${CROSS_AR}")
    set(ENV{RANLIB} "${CROSS_RANLIB}")
    execute_process(
        COMMAND ./configure --static
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/zlib-1.2.13"
    )
    execute_process(
        COMMAND make -j${NPROC}
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/zlib-1.2.13"
    )
    execute_process(
        COMMAND cp -f libz.a ../libs
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/zlib-1.2.13"
    )
    # Copy zlib headers
    execute_process(
        COMMAND cp -f zlib.h zconf.h ../include/
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/thirdparty/zlib-1.2.13"
    )
endif(THIRDPARTY_STATIC_BUILD)

