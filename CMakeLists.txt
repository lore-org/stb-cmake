# ---- Project Info ----

cmake_minimum_required(VERSION 3.10)
project(stb VERSION 0.2.1)

# ---- Set modifiable variables ----

set(STB_COMMIT f1c79c0 CACHE STRING "Commit of stb to pull. This is only designed for backwards compatibility, not forwards.")

set(CPM_VERSION 0.42.0 CACHE STRING "Version of CPM to download.")
set(CPM_HASH eed8973c5595c72d9ba785a5472e9750 CACHE STRING "MD5 hash of the specific version of CPM.")

# ---- Handle CPM ----

set(_cpm_file "${CMAKE_BINARY_DIR}/cmake/get_cpm_${CPM_VERSION}.cmake")
if (CPM_SOURCE_CACHE)
    set(_cpm_file "${CPM_SOURCE_CACHE}/cpm/get_cpm_${CPM_VERSION}.cmake")
elseif(ENV{CPM_SOURCE_CACHE})
    set(_cpm_file "$ENV{CPM_SOURCE_CACHE}/cpm/get_cpm_${CPM_VERSION}.cmake")
endif()

get_filename_component(_cpm_file "${_cpm_file}" ABSOLUTE)

if (EXISTS "${_cpm_file}")
    file(MD5 "${_cpm_file}" CURRENT_CPM_HASH)

    if (CURRENT_CPM_HASH STREQUAL CPM_HASH)
        set(CURRENT_CPM_VALID TRUE)
        message(STATUS "Found existing CPM v${CPM_VERSION} (MD5: ${CPM_HASH})")
    endif()
endif()

if (NOT CURRENT_CPM_VALID)
    message(STATUS "Downloading CPM v${CPM_VERSION} (MD5: ${CPM_HASH})")
    file(DOWNLOAD
        "https://github.com/cpm-cmake/CPM.cmake/releases/download/v${CPM_VERSION}/get_cpm.cmake"
        "${_cpm_file}"
        EXPECTED_HASH MD5=${CPM_HASH}
        SHOW_PROGRESS
    )
endif()

include("${_cpm_file}")

# ---- Download STB ----

CPMAddPackage(
    stb
    GIT_REPOSITORY https://github.com/nothings/stb.git
    GIT_TAG ${STB_COMMIT}
    DOWNLOAD_ONLY
)

# ---- Include simple STB libs ----

set(STB_DIR "${CMAKE_CURRENT_BINARY_DIR}/.stb")

set(EMPTY_SRC "${STB_DIR}/empty.c")
file(WRITE "${EMPTY_SRC}" "")

macro(format_library LIBRARY_NAME)
    set(STB_LIB_NAME stb_${LIBRARY_NAME})

    target_compile_definitions(${STB_LIB_NAME} PRIVATE STB_${UPPER_LIB_NAME}_IMPLEMENTATION)
    add_library(stb::${LIBRARY_NAME} ALIAS ${STB_LIB_NAME})

    list(APPEND STB_LIB_LIST stb::${LIBRARY_NAME})
endmacro()

function(add_simple_library LIBRARY_NAME)
    set(STB_LIB_NAME stb_${LIBRARY_NAME})
    string(TOUPPER ${LIBRARY_NAME} UPPER_LIB_NAME)

    if (EXISTS "${stb_SOURCE_DIR}/${STB_LIB_NAME}.h") # Library may not exist on older versions
        add_library                 (${STB_LIB_NAME} STATIC "${EMPTY_SRC}")
        target_include_directories  (${STB_LIB_NAME} INTERFACE "${stb_SOURCE_DIR}")
        
        format_library(${LIBRARY_NAME})
    endif()
endfunction()

add_simple_library(c_lexer)
add_simple_library(connected_components)
add_simple_library(divide)
add_simple_library(ds)
add_simple_library(dxt)
add_simple_library(easy_font)
add_simple_library(herringbone_wang_tile)
add_simple_library(hexwave)
add_simple_library(image)
add_simple_library(image_resize)
add_simple_library(image_resize2)
add_simple_library(image_write)
add_simple_library(include)
add_simple_library(leakcheck)
add_simple_library(perlin)
add_simple_library(rect_pack)
add_simple_library(sprintf)
add_simple_library(textedit)
add_simple_library(tilemap_editor)
add_simple_library(truetype)
add_simple_library(voxel_render)

# ---- Include and fix missing libs ----

# ------ stb_vorbis.c ------

if (EXISTS "${stb_SOURCE_DIR}/stb_vorbis.c")
    add_library                 (stb_vorbis STATIC "${stb_SOURCE_DIR}/stb_vorbis.c")
    target_include_directories  (stb_vorbis PUBLIC "${stb_SOURCE_DIR}")
    target_compile_definitions  (stb_vorbis INTERFACE STB_VORBIS_HEADER_ONLY)
    
    format_library(vorbis)
endif()

# ------ stb_image_resize2.h ------

if (EXISTS "${stb_SOURCE_DIR}/stb_image_resize2.h")
    target_compile_definitions(stb_image_resize2 PRIVATE STB_IMAGE_RESIZE_IMPLEMENTATION)
endif()

# ---- Add stb::all ----

add_library(stb_all INTERFACE)
target_link_libraries(stb_all INTERFACE ${STB_LIB_LIST})

add_library(stb::all ALIAS stb_all)
add_library(stb ALIAS stb_all)
