#
# Copyright (C) 2010-2019 (see AUTHORS file for a list of contributors)
#
# This file is part of GNSS-SDR.
#
# GNSS-SDR is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# GNSS-SDR is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with GNSS-SDR. If not, see <https://www.gnu.org/licenses/>.
#

########################################################################
# Setup profiler
########################################################################
if(MSVC)
    include_directories(${PROJECT_SOURCE_DIR}/cmake/msvc)
endif()

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_BINARY_DIR}
    ${PROJECT_SOURCE_DIR}/include
    ${PROJECT_BINARY_DIR}/include
    ${PROJECT_SOURCE_DIR}/lib
    ${PROJECT_BINARY_DIR}/lib
    ${Boost_INCLUDE_DIRS}
)


if(ORC_FOUND)
    set(orc_lib ${ORC_LIBRARIES})
else()
    set(orc_lib "")
endif()


# allow 'large' files in 32 bit builds
if(UNIX)
    add_definitions(-D_LARGEFILE_SOURCE
        -D_FILE_OFFSET_BITS=64
        -D_LARGE_FILES
    )
endif()


# MAKE volk_gnsssdr_profile
add_executable(volk_gnsssdr_profile
    ${CMAKE_CURRENT_SOURCE_DIR}/volk_gnsssdr_profile.cc
    ${PROJECT_SOURCE_DIR}/lib/qa_utils.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/volk_gnsssdr_option_helpers.cc
)

if(${FILESYSTEM_FOUND})
    add_definitions(-DHAS_STD_FILESYSTEM=1)
    if(${find_experimental})
        add_definitions(-DHAS_STD_FILESYSTEM_EXPERIMENTAL=1)
    endif()
    target_link_libraries(volk_gnsssdr_profile PRIVATE std::filesystem)
endif()

if(ENABLE_STATIC_LIBS)
    target_link_libraries(volk_gnsssdr_profile PUBLIC volk_gnsssdr_static ${Boost_LIBRARIES} ${orc_lib})
else()
    target_link_libraries(volk_gnsssdr_profile PUBLIC volk_gnsssdr ${Boost_LIBRARIES} ${orc_lib})
    add_dependencies(volk_gnsssdr_profile volk_gnsssdr)
endif()


if(ENABLE_STRIP)
    if((CMAKE_CXX_COMPILER_ID STREQUAL "GNU") AND NOT WIN32)
        set_target_properties(volk_gnsssdr_profile
            PROPERTIES LINK_FLAGS "-s")
    endif()
endif()


install(
    TARGETS volk_gnsssdr_profile
    DESTINATION bin
    COMPONENT "volk_gnsssdr"
)


# MAKE volk_gnsssdr-config-info
add_executable(volk_gnsssdr-config-info volk_gnsssdr-config-info.cc ${CMAKE_CURRENT_SOURCE_DIR}/volk_gnsssdr_option_helpers.cc)


if(ENABLE_STATIC_LIBS)
    target_link_libraries(volk_gnsssdr-config-info volk_gnsssdr_static ${orc_lib})
else()
    target_link_libraries(volk_gnsssdr-config-info volk_gnsssdr ${orc_lib})
    add_dependencies(volk_gnsssdr-config-info volk_gnsssdr)
endif()

if(ENABLE_STRIP)
    if((CMAKE_CXX_COMPILER_ID STREQUAL "GNU") AND NOT WIN32)
        set_target_properties(volk_gnsssdr-config-info
            PROPERTIES LINK_FLAGS "-s")
    endif()
endif()

install(
    TARGETS volk_gnsssdr-config-info
    DESTINATION bin
    COMPONENT "volk_gnsssdr"
)


# Launch volk_gnsssdr_profile if requested to do so
if(ENABLE_PROFILING)
    if(DEFINED VOLK_CONFIGPATH)
        set( VOLK_CONFIG_ARG "-p${VOLK_CONFIGPATH}" )
        set( VOLK_CONFIG "${VOLK_CONFIGPATH}/volk_gnsssdr_config" )
    endif()
    add_custom_command(OUTPUT ${VOLK_CONFIG}
        COMMAND volk_gnsssdr_profile "${VOLK_CONFIG_ARG}"
        DEPENDS volk_gnsssdr_profile
        COMMENT "Launching profiler, this may take a few minutes..."
    )
    add_custom_target(volk-gnsssdr-profile-run ALL DEPENDS ${VOLK_CONFIG})
endif()
