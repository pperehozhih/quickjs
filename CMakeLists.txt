cmake_minimum_required(VERSION 3.10)
project(quickjs C)

option(QJS_BUILD_STATIC "Build static library" ON)
option(QJS_BUILD_SHARED "Build shared library" OFF)
option(QJS_BUILD_TOOLS "Build CLI tools (qjs, qjsc, repl)" ON)

# читаем версию
set(VERSION_FILE "${CMAKE_SOURCE_DIR}/VERSION")
if(EXISTS "${VERSION_FILE}")
    file(READ "${VERSION_FILE}" QJS_VERSION_RAW)
    string(STRIP "${QJS_VERSION_RAW}" QJS_VERSION)
else()
    set(QJS_VERSION "unknown")
endif()

set(SOURCES
    quickjs.c
    cutils.c
    libunicode.c
    libregexp.c
    dtoa.c
    quickjs-libc.c
)

set(HEADERS
    quickjs.h
    cutils.h
    libunicode.h
    libregexp.h
    dtoa.h
    quickjs-libc.h
)

if(QJS_BUILD_STATIC)
    add_library(quickjs STATIC ${SOURCES} ${HEADERS})
    target_compile_definitions(quickjs PRIVATE _GNU_SOURCE CONFIG_VERSION=\"${QJS_VERSION}\")
    target_link_libraries(quickjs m pthread dl)
endif()

if(QJS_BUILD_SHARED)
    add_library(quickjs_shared SHARED ${SOURCES} ${HEADERS})
    target_compile_definitions(quickjs_shared PRIVATE JS_SHARED_LIBRARY _GNU_SOURCE CONFIG_VERSION=\"${QJS_VERSION}\")
    target_link_libraries(quickjs_shared m pthread dl)
endif()

if(QJS_BUILD_TOOLS)
    add_executable(qjsc qjsc.c)
    target_compile_definitions(qjsc PRIVATE CONFIG_VERSION=\"${QJS_VERSION}\")
    if(QJS_BUILD_STATIC)
        target_link_libraries(qjsc quickjs)
    elseif(QJS_BUILD_SHARED)
        target_link_libraries(qjsc quickjs_shared)
    endif()

    add_custom_command(
        OUTPUT ${CMAKE_BINARY_DIR}/repl.c
        COMMAND qjsc -s -c -o ${CMAKE_BINARY_DIR}/repl.c -m ${CMAKE_SOURCE_DIR}/repl.js
        DEPENDS qjsc ${CMAKE_SOURCE_DIR}/repl.js
    )

    add_custom_target(generate_repl DEPENDS ${CMAKE_BINARY_DIR}/repl.c)

    add_executable(qjs qjs.c ${CMAKE_BINARY_DIR}/repl.c)
    add_dependencies(qjs generate_repl)
    target_compile_definitions(qjs PRIVATE CONFIG_VERSION=\"${QJS_VERSION}\")
    if(QJS_BUILD_STATIC)
        target_link_libraries(qjs quickjs)
    elseif(QJS_BUILD_SHARED)
        target_link_libraries(qjs quickjs_shared)
    endif()
endif()

include(GNUInstallDirs)

if(QJS_BUILD_STATIC)
    install(TARGETS quickjs
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    )
endif()

if(QJS_BUILD_SHARED)
    install(TARGETS quickjs_shared
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    )
endif()

install(FILES
    quickjs.h
    quickjs-libc.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/quickjs
)

if(QJS_BUILD_TOOLS)
    install(TARGETS qjs qjsc
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
endif()
