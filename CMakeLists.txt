project(SoftRaster)

include_directories(${CMAKE_CURRENT_SOURCE_DIR})

file(GLOB INC ${CMAKE_CURRENT_SOURCE_DIR}/*.h)
file(GLOB SRC ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp)

add_executable(SoftRaster
    "${INC}"
    "${SRC}"
)

set_target_properties(SoftRaster PROPERTIES VS_DEBUGGER_COMMAND_ARGUMENTS "${CMAKE_CURRENT_SOURCE_DIR}/obj/african_head.obj")
source_group("Include" FILES ${INC})
source_group("Source" FILES ${SRC})

