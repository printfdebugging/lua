cmake_minimum_required(VERSION 4.0)

project(lua
   VERSION         5.4
   DESCRIPTION     "Lua"
   HOMEPAGE_URL    "https://github.com/lua/lua"
   LANGUAGES       C
)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)
include(CheckLibraryExists)

option(LUA_ENABLE_TESTS "Build the internal test/debug harness (ltests.c)" OFF)

set(CMAKE_C_STANDARD 23)
set(CMAKE_C_STANDARD_REQUIRED ON)

set(LUA_LIB_NAME lua)
set(LUA ${CMAKE_CURRENT_LIST_DIR}/lua5.4)

set(LUA_CORE
   lapi.c lcode.c lctype.c ldebug.c ldo.c ldump.c lfunc.c lgc.c llex.c
   lmem.c lobject.c lopcodes.c lparser.c lstate.c lstring.c ltable.c
   ltm.c lundump.c lvm.c lzio.c
)

set(LUA_STDLIB
   lauxlib.c lbaselib.c lcorolib.c ldblib.c liolib.c lmathlib.c
   loadlib.c loslib.c lstrlib.c ltablib.c lutf8lib.c linit.c
)

set(LUA_PUBLIC_HEADERS
   lua.h luaconf.h lualib.h lauxlib.h
)

list(TRANSFORM LUA_CORE           PREPEND ${LUA}/)
list(TRANSFORM LUA_STDLIB         PREPEND ${LUA}/)
list(TRANSFORM LUA_PUBLIC_HEADERS PREPEND ${LUA}/)

if(EXISTS ${LUA}/lua.hpp)
   list(APPEND LUA_PUBLIC_HEADERS ${LUA}/lua.hpp)
endif()

set(LUA_WARNINGS)

if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
   set(LUA_WARNINGS
      -Wfatal-errors
      -Wextra
      -Wshadow
      -Wundef
      -Wwrite-strings
      -Wredundant-decls
      -Wdisabled-optimization
      -Wdouble-promotion
      -Wmissing-declarations
      -Wno-unused-result
   )
endif()

add_library(lua_lib STATIC
   ${LUA_CORE}
   ${LUA_STDLIB}
)

add_library(lua::lua ALIAS lua_lib)

set_target_properties(lua_lib PROPERTIES
   OUTPUT_NAME ${LUA_LIB_NAME}
)

target_include_directories(lua_lib PUBLIC
   $<BUILD_INTERFACE:${LUA}>
   $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/lua>
)

target_compile_options(lua_lib PRIVATE ${LUA_WARNINGS})

check_library_exists(m sin "" LUA_HAVE_LIBM)

if(LUA_HAVE_LIBM)
   target_link_libraries(lua_lib PUBLIC m)
endif()

if(UNIX AND NOT APPLE)
   target_compile_definitions(lua_lib PUBLIC LUA_USE_LINUX)
   target_link_libraries(lua_lib PUBLIC ${CMAKE_DL_LIBS})
elseif(APPLE)
   target_compile_definitions(lua_lib PUBLIC LUA_USE_MACOSX)
   target_link_libraries(lua_lib PUBLIC ${CMAKE_DL_LIBS})
endif()

if(LUA_ENABLE_TESTS)
   target_sources(lua_lib PRIVATE ${LUA}/ltests.c)
   target_compile_definitions(lua_lib PUBLIC LUA_USER_H="ltests.h")
endif()

add_executable(lua_exe
   ${LUA}/lua.c
)

set_target_properties(lua_exe PROPERTIES
   OUTPUT_NAME lua
)

target_compile_options(lua_exe PRIVATE ${LUA_WARNINGS})

target_link_libraries(lua_exe
   PRIVATE
   lua_lib
)

if(EXISTS ${LUA}/luac.c)
   add_executable(luac_exe
      ${LUA}/luac.c
   )

   set_target_properties(luac_exe PROPERTIES
      OUTPUT_NAME luac
   )

   target_compile_options(luac_exe PRIVATE ${LUA_WARNINGS})

   target_link_libraries(luac_exe
      PRIVATE
      lua_lib
   )
endif()

set(LUA_INSTALL_TARGETS lua_lib lua_exe)

if(TARGET luac_exe)
   list(APPEND LUA_INSTALL_TARGETS luac_exe)
endif()

configure_file(
   ${CMAKE_CURRENT_SOURCE_DIR}/lua.pc.in
   ${CMAKE_CURRENT_BINARY_DIR}/lua.pc @ONLY
)

write_basic_package_version_file(
   ${CMAKE_CURRENT_BINARY_DIR}/luaConfigVersion.cmake
   VERSION       ${PROJECT_VERSION}
   COMPATIBILITY SameMajorVersion
)

install(
   FILES ${CMAKE_CURRENT_BINARY_DIR}/lua.pc
   DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)

install(
   FILES ${LUA_PUBLIC_HEADERS}
   DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/lua
)

install(
   TARGETS ${LUA_INSTALL_TARGETS}
   EXPORT lua_targets
   LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
   ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
   RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(
   EXPORT      lua_targets
   FILE        luaConfig.cmake
   NAMESPACE   lua::
   DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/lua
)

install(
   FILES ${CMAKE_CURRENT_BINARY_DIR}/luaConfigVersion.cmake
   DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/lua
)
