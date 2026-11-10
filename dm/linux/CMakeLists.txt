cmake_minimum_required(VERSION 3.22)
project(dm_linux C CXX ASM)
add_compile_definitions(_GNU_SOURCE)

set(DM ${CMAKE_CURRENT_SOURCE_DIR}/..)
set(HOST ${DM}/host)
get_filename_component(DM_REPO_ROOT ${DM}/.. ABSOLUTE)

if(NOT DM_GEN_DIR OR NOT EXISTS "${DM_GEN_DIR}/dm_vault_tab.h")
  message(FATAL_ERROR "Set -DDM_GEN_DIR to the private generated-input directory")
endif()
foreach(required dm_hud_exp.h dm_bright_levels.inc dm_touch_native_art.c)
  if(NOT EXISTS "${DM_GEN_DIR}/${required}")
    message(FATAL_ERROR "Missing ${DM_GEN_DIR}/${required}; generate game-derived inputs first")
  endif()
endforeach()

find_package(Python3 REQUIRED COMPONENTS Interpreter)
find_package(Threads REQUIRED)
find_package(Vulkan REQUIRED)
find_package(PkgConfig REQUIRED)
pkg_check_modules(SDL2 REQUIRED IMPORTED_TARGET sdl2)

add_subdirectory(${DM}/gfx/vulkan ${CMAKE_CURRENT_BINARY_DIR}/vulkan)

# Keep the shared host source list in sync with dm/host/Makefile.
set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS
  ${HOST}/Makefile ${DM}/android/tools/host_sources.py)
execute_process(
  COMMAND ${Python3_EXECUTABLE} ${DM}/android/tools/host_sources.py
    ${HOST}/Makefile ${CMAKE_CURRENT_BINARY_DIR}/host_sources.cmake
  RESULT_VARIABLE source_list_result)
if(NOT source_list_result EQUAL 0)
  message(FATAL_ERROR "Could not read dm/host/Makefile source list")
endif()
include(${CMAKE_CURRENT_BINARY_DIR}/host_sources.cmake)
file(GLOB HLE_SRCS CONFIGURE_DEPENDS ${HOST}/hle_*.c)
file(GLOB GFX_SRCS CONFIGURE_DEPENDS ${DM}/gfx/*.c ${DM}/gfx/nvn/*.c ${DM}/gfx/tex/*.c)

set(ASM_SRCS)
foreach(source ${HOST}/entry.S ${DM}/gfx/nvn/nvn_entry.S ${DM}/gfx/nvn/nvn_thunks.S)
  get_filename_component(name ${source} NAME)
  set(output ${CMAKE_CURRENT_BINARY_DIR}/asm/${name})
  add_custom_command(OUTPUT ${output}
    COMMAND ${Python3_EXECUTABLE} ${DM}/android/tools/asm_elf.py ${source} ${output}
    DEPENDS ${source} ${DM}/android/tools/asm_elf.py VERBATIM)
  list(APPEND ASM_SRCS ${output})
endforeach()

add_library(dmcore STATIC
  ${HOST_SRCS} ${HLE_SRCS} ${GFX_SRCS} ${LIBDEFLATE_SRCS} ${ASM_SRCS}
  ${DM}/gfx/vulkan/nvn_vk.c ${DM_GEN_DIR}/dm_touch_native_art.c
  ${CMAKE_CURRENT_SOURCE_DIR}/front_linux.c ${CMAKE_CURRENT_SOURCE_DIR}/vulkan_present.cpp
  ${DM}/android/compat/clonefile.c ${DM}/android/compat/digest.c
  ${DM}/android/compat/mach_thread.c ${DM}/android/compat/mach_symbols.c)
# Guest threads run on separately allocated Horizon stacks and return to the
# pthread stack with siglongjmp. glibc's fortified __longjmp_chk cannot account
# for that intentional stack switch and aborts depending on the stack addresses.
set_source_files_properties(${HOST}/hle_thread.c PROPERTIES
  COMPILE_OPTIONS "-U_FORTIFY_SOURCE;-D_FORTIFY_SOURCE=0")
target_include_directories(dmcore PUBLIC
  ${HOST} ${DM}/gfx/nvn/include ${DM}/gfx/vulkan ${DM_GEN_DIR}
  ${DM}/ios/Sources ${DM}/android/compat/include)
target_compile_options(dmcore PUBLIC
  $<$<COMPILE_LANGUAGE:C>:-O1;-g;-Wall;-Wno-unused-function;-fno-strict-aliasing;-std=gnu17;-include${DM}/android/compat/include/dm_android_compat.h>
  $<$<COMPILE_LANGUAGE:CXX>:-O1;-g;-Wall;-Wno-unused-function;-fno-strict-aliasing;-std=gnu++17>)
target_compile_definitions(dmcore PUBLIC DM_LINUX=1)
target_link_libraries(dmcore PUBLIC dm_vulkan_engine Threads::Threads Vulkan::Vulkan
  PkgConfig::SDL2 m dl)

add_executable(d2r-mobile ${HOST}/main.c)
target_link_libraries(d2r-mobile PRIVATE -Wl,--whole-archive dmcore -Wl,--no-whole-archive)
target_link_options(d2r-mobile PRIVATE -Wl,--no-undefined -Wl,-z,nostart-stop-gc -Wl,--export-dynamic)
set_target_properties(d2r-mobile PROPERTIES OUTPUT_NAME d2r-mobile)
install(TARGETS d2r-mobile RUNTIME DESTINATION bin)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/d2r-mobile.desktop
  DESTINATION share/applications)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/d2r-mobile.svg
  DESTINATION share/icons/hicolor/scalable/apps)
