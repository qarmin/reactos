
include(../ntos.cmake)

include_directories(${REACTOS_SOURCE_DIR}/ntoskrnl)
spec2def(ntkrnlmp.exe ../ntoskrnl.spec)

if(MSVC_IDE)
    set_source_files_properties(${NTKRNLMP_ASM_SOURCE} PROPERTIES COMPILE_DEFINITIONS "CONFIG_SMP")
endif()

add_asm_files(ntkrnlmp_asm ${NTKRNLMP_ASM_SOURCE})

list(APPEND NTKRNLMP_PCH_SKIP_SOURCE
    ${REACTOS_SOURCE_DIR}/ntoskrnl/guid.c)

add_executable(ntkrnlmp
    ${ntkrnlmp_asm}
    ${NTKRNLMP_SOURCE}
    ${NTKRNLMP_PCH_SKIP_SOURCE}
    ${REACTOS_SOURCE_DIR}/ntoskrnl/ntoskrnl.rc
    ${CMAKE_CURRENT_BINARY_DIR}/ntkrnlmp.def)
set_property(TARGET ntkrnlmp PROPERTY ENABLE_EXPORTS TRUE)

target_compile_definitions(ntkrnlmp PRIVATE CONFIG_SMP)

if(ARCH STREQUAL "i386")
    set_entrypoint(ntkrnlmp KiSystemStartup 4)
else()
    set_entrypoint(ntkrnlmp KiSystemStartup)
endif()
set_subsystem(ntkrnlmp native)

if(MSVC)
    set_image_base(ntkrnlmp 0x00400000)
    add_target_link_flags(ntkrnlmp "/SECTION:.rsrc,!DP") #Accessed from bugcheck code
else()
    set_image_base(ntkrnlmp 0x80800000)
endif()

# dynamic analysis switches
if(STACK_PROTECTOR)
    target_sources(ntkrnlmp PRIVATE $<TARGET_OBJECTS:gcc_ssp_nt>)
endif()

if(RUNTIME_CHECKS)
    target_link_libraries(ntkrnlmp runtmchk)
endif()

target_link_libraries(ntkrnlmp cportlib csq ${PSEH_LIB} arbiter cmlib ntlsalib rtl ${ROSSYM_LIB} libcntpr wdmguid ioevent)
add_importlibs(ntkrnlmp hal kdcom bootvid)
add_pch(ntkrnlmp ${REACTOS_SOURCE_DIR}/ntoskrnl/include/ntoskrnl.h "${NTKRNLMP_PCH_SKIP_SOURCE}")
add_dependencies(ntkrnlmp psdk bugcodes asm)
add_cd_file(TARGET ntkrnlmp DESTINATION reactos/system32 NO_CAB FOR all)
