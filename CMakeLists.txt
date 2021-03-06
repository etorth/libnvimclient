#=======================================================================================
#
#        Filename: CMakeLists.txt
#         Created: 12/25/2018 13:19:07
#     Description:
#
#         Version: 1.0
#        Revision: none
#        Compiler: cmake
#
#          Author: ANHONG
#           Email: anhonghe@gmail.com
#    Organization: USTC
#
#=======================================================================================

PROJECT(LIBNVC)
CMAKE_MINIMUM_REQUIRED(VERSION 3.12)

SET(CMAKE_CXX_STANDARD 20)
SET(CMAKE_CXX_STANDARD_REQUIRED ON)

# add flags to release mode
# by default won't use release mode
SET(CMAKE_CXX_FLAGS_RELEASE "$ENV{CXXFLAGS}")
SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3")
SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -fno-strict-aliasing")
SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Wall -Wfatal-errors -Wextra -Wunused -Werror -f")

# always compile libnvc in debug mode
# for trace message use LIBNVC_DEBUG_ARG_LIST to enable
# SET(CMAKE_BUILD_TYPE "Debug")
SET(CMAKE_CXX_FLAGS_DEBUG "$ENV{CXXFLAGS}")
SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0")

# enable check for asan/usan/tsan
# keep in mind that these three can't be set at the same time

OPTION(LIBNVC_ENABLE_ASAN "Enable ASAN" OFF)
OPTION(LIBNVC_ENABLE_USAN "Enable USAN" OFF)
OPTION(LIBNVC_ENABLE_TSAN "Enable TSAN" OFF)

IF(LIBNVC_ENABLE_ASAN)
    MESSAGE(STATUS "ASAN enabled")
    SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=address")
ENDIF()

IF(LIBNVC_ENABLE_USAN)
    MESSAGE(STATUS "USAN enabled")
    SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=undefined")
ENDIF()

IF(LIBNVC_ENABLE_TSAN)
    MESSAGE(STATUS "TSAN enabled")
    SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=thread")
ENDIF()

SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g3 -ggdb3 -pedantic")
SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fno-strict-aliasing")
SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -Wall -Wextra -Wunused -Werror")

SET(LIBNVC_3RD_PARTY_DIR "${CMAKE_CURRENT_BINARY_DIR}/3rdparty")

# set additional module path
LIST(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules")

FILE(GLOB LIBNVC_DOWNLOAD_SUBDIR "${CMAKE_CURRENT_SOURCE_DIR}/cmake/download/*.cmake")
FOREACH(LIBNVC_DOWNLOAD ${LIBNVC_DOWNLOAD_SUBDIR})
    INCLUDE(${LIBNVC_DOWNLOAD})
ENDFOREACH()

FILE(GLOB LIBNVC_DEPS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/cmake/deps/*.cmake")
FOREACH(LIBNVC_DEP ${LIBNVC_DEPS_DIR})
    INCLUDE(${LIBNVC_DEP})
ENDFOREACH()

SET(CMAKE_SKIP_RPATH TRUE)

SET(LIBNVC_JINJA2_INPUT_DIR  "${CMAKE_SOURCE_DIR}/template")
SET(LIBNVC_JINJA2_OUTPUT_DIR "${CMAKE_CURRENT_BINARY_DIR}/src")

# include all generated files by jinja2
# seems can't use aux_source_directory() since it's empty now ?
FILE(GLOB JINJA2_ALL_TEMPLATE_SRC "${LIBNVC_JINJA2_INPUT_DIR}/*.[hc]pp")

# ADD_CUSTOM_TARGET(jinja2_proc ALL
#     python3 ${CMAKE_SOURCE_DIR}/nvim_api.py
#          -t ${LIBNVC_JINJA2_INPUT_DIR}
#          -o ${LIBNVC_JINJA2_OUTPUT_DIR}
#     COMMENT "Generate nvim server msgpack-rpc api..."
#     DEPENDS ${JINJA2_ALL_TEMPLATE_SRC}
# )

SET(JINJA2_SRC "")
FOREACH(JINJA2_CURR_TEMPLATE_SRC ${JINJA2_ALL_TEMPLATE_SRC})
    GET_FILENAME_COMPONENT(JINJA2_CURR_SRC ${JINJA2_CURR_TEMPLATE_SRC} NAME)
    SET(JINJA2_SRC ${JINJA2_SRC} ${LIBNVC_JINJA2_OUTPUT_DIR}/${JINJA2_CURR_SRC})
ENDFOREACH()

ADD_CUSTOM_COMMAND(OUTPUT ${JINJA2_SRC}
    COMMAND python3 ${CMAKE_SOURCE_DIR}/nvim_api.py -t ${LIBNVC_JINJA2_INPUT_DIR} -o ${LIBNVC_JINJA2_OUTPUT_DIR}
    DEPENDS ${CMAKE_SOURCE_DIR}/nvim_api.py ${JINJA2_ALL_TEMPLATE_SRC}
    COMMENT "Generate nvim server msgpack-rpc api..."
    VERBATIM 
)

ADD_CUSTOM_TARGET(jinja2_proc DEPENDS ${JINJA2_SRC})

AUX_SOURCE_DIRECTORY(${MPACK_SRC_DIR}   MPACK_SRC)
AUX_SOURCE_DIRECTORY(${EXTFUNC_SRC_DIR} EXTFUNC_SRC)

ADD_LIBRARY(nvc ${JINJA2_SRC} ${MPACK_SRC} ${EXTFUNC_SRC})
ADD_DEPENDENCIES(nvc jinja2_proc asio reproc mirror_utf8cpp)
TARGET_LINK_LIBRARIES(nvc PRIVATE reproc++)

TARGET_INCLUDE_DIRECTORIES(nvc PUBLIC "${MPACK_INCLUDE_DIRS}")
TARGET_INCLUDE_DIRECTORIES(nvc PUBLIC "${EXTFUNC_INCLUDE_DIRS}")
TARGET_INCLUDE_DIRECTORIES(nvc PUBLIC "${LIBNVC_JINJA2_OUTPUT_DIR}")

INSTALL(TARGETS nvc DESTINATION lib)
INSTALL(FILES ${LIBNVC_JINJA2_OUTPUT_DIR}/libnvc.hpp DESTINATION include)
