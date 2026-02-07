# -------------------------------------------------
#  CMakeLists.txt
#  meson.build conversion to CMake by gpt and manual correction
# -------------------------------------------------
cmake_minimum_required(VERSION 3.20)
project(inih LANGUAGES C CXX)
message(STATUS "inih project")
# ----------------------------------------------------------------------
# 1. Options (Meson_options.txt)
# ----------------------------------------------------------------------
message(STATUS "Options init")
option(DISTRO_INSTALL "install shared libs, headers and pkg‑config entries" ON)
option(WITH_INIREADER "compile and (if selected) install INIReader" ON)

option(MULTI_LINE_ENTRIES "support for multi‑line entries" ON)
option(UTF8_BOM "allow a UTF‑8 BOM sequence at the start of INI files" ON)
option(INLINE_COMMENTS "allow inline comments with the comment prefix character" ON)
set(INLINE_COMMENT_PREFIX ";" CACHE STRING "character(s) to start an inline comment")
set(START_OF_LINE_COMMENT_PREFIX ";#" CACHE STRING "character(s) to start a comment at line start")

option(ALLOW_NO_VALUE "allow name with no value" OFF)
option(STOP_ON_FIRST_ERROR "stop parsing after an error" OFF)
option(REPORT_LINE_NUMBERS "report line number on ini_handler callback" OFF)
option(CALL_HANDLER_ON_NEW_SECTION "call the handler each time a new section is encountered" OFF)
option(USE_HEAP "allocate memory on the heap using malloc" OFF)

set(MAX_LINE_LENGTH 200 CACHE STRING "maximum line length in bytes")
set(INITIAL_MALLOC_SIZE 200 CACHE STRING "initial malloc size in bytes (when using the heap)")
option(ALLOW_REALLOC "allow initial malloc size to grow to max line length" OFF)

option(BUILD_TESTS "build the test suite (noisy)" ON)

# ----------------------------------------------------------------------
# 2. Preparing preprocessor's flags
# ----------------------------------------------------------------------
set(ARG_STATIC "")
set(EXTRA_ARGS "")

if(NOT DISTRO_INSTALL)
    message(STATUS "Not distro install. Adding defines.")
    if(NOT MULTI_LINE_ENTRIES)
        list(APPEND ARG_STATIC "-DINI_ALLOW_MULTILINE=0")
    endif()
    if(NOT UTF8_BOM)
        list(APPEND ARG_STATIC "-DINI_ALLOW_BOM=0")
    endif()
    if(NOT INLINE_COMMENTS)
        list(APPEND ARG_STATIC "-DINI_ALLOW_INLINE_COMMENTS=0")
    endif()
    if(NOT INLINE_COMMENT_PREFIX STREQUAL ";")
        list(APPEND ARG_STATIC "-DINI_INLINE_COMMENT_PREFIXES=\"${INLINE_COMMENT_PREFIX}\"")
    endif()
    if(NOT START_OF_LINE_COMMENT_PREFIX STREQUAL ";#")
        list(APPEND ARG_STATIC "-DINI_START_COMMENT_PREFIXES=\"${START_OF_LINE_COMMENT_PREFIX}\"")
    endif()
    if(ALLOW_NO_VALUE)
        list(APPEND ARG_STATIC "-DINI_ALLOW_NO_VALUE=1")
    endif()
    if(STOP_ON_FIRST_ERROR)
        list(APPEND ARG_STATIC "-DINI_STOP_ON_FIRST_ERROR=1")
    endif()
    if(REPORT_LINE_NUMBERS)
        list(APPEND ARG_STATIC "-DINI_HANDLER_LINENO=1")
    endif()
    if(CALL_HANDLER_ON_NEW_SECTION)
        list(APPEND ARG_STATIC "-DINI_CALL_HANDLER_ON_NEW_SECTION=1")
    endif()
    if(USE_HEAP)
        list(APPEND ARG_STATIC "-DINI_USE_STACK=0")
    endif()
    if(NOT MAX_LINE_LENGTH STREQUAL "200")
        list(APPEND ARG_STATIC "-DINI_MAX_LINE=${MAX_LINE_LENGTH}")
    endif()
    if(NOT INITIAL_MALLOC_SIZE STREQUAL "200")
        list(APPEND ARG_STATIC "-DINI_INITIAL_ALLOC=${INITIAL_MALLOC_SIZE}")
    endif()
    if(ALLOW_REALLOC)
        list(APPEND ARG_STATIC "-DINI_ALLOW_REALLOC=1")
    endif()
endif()

# Windows‑specific handling (shared‑lib flag)
if(WIN32)
    message(STATUS "Windows specific handling (shared-lib flag)")
    if(BUILD_SHARED_LIBS)
        list(APPEND EXTRA_ARGS "-DINI_SHARED_LIB")
        add_compile_definitions(INI_SHARED_LIB_BUILDING)
    endif()
endif()

# ----------------------------------------------------------------------
# 3. library inih (C)
# ----------------------------------------------------------------------
add_library(inih STATIC ini.c)               # будет SHARED если -DBUILD_SHARED_LIBS=ON
target_include_directories(inih PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_compile_options(inih PRIVATE ${ARG_STATIC} ${EXTRA_ARGS})

set_target_properties(inih PROPERTIES
        VERSION 0.0.0
        SOVERSION 0
        POSITION_INDEPENDENT_CODE ON
        C_VISIBILITY_PRESET hidden
        CXX_VISIBILITY_PRESET hidden
)

# ---------- 3.1. dependency‑target (interface) ----------
add_library(inih_interface INTERFACE)               #
add_library(inih::inih ALIAS inih_interface)        # alias‑target, for users

target_include_directories(inih_interface INTERFACE ${CMAKE_CURRENT_SOURCE_DIR})
target_link_libraries(inih_interface INTERFACE inih)   # link to real lib
target_compile_options(inih_interface INTERFACE ${ARG_STATIC} ${EXTRA_ARGS})

# ----------------------------------------------------------------------
# 4. Install (if distro mode enabled)
# ----------------------------------------------------------------------
if(DISTRO_INSTALL)
    message(STATUS "Distro install -- TRUE")
    install(TARGETS inih
            ARCHIVE DESTINATION lib
            LIBRARY DESTINATION lib
            RUNTIME DESTINATION bin
    )
    install(FILES ini.h DESTINATION include)

    # ---- pkg‑config generation ----
    configure_file(pkg-config.in inih.pc @ONLY)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/inih.pc DESTINATION lib/pkgconfig)
endif()

# ----------------------------------------------------------------------
# 5. Tests
# ----------------------------------------------------------------------
if(BUILD_TESTS)
    message(STATUS "Build tests -- TRUE")
    enable_testing()
    add_subdirectory(tests)
endif()

# ----------------------------------------------------------------------
# 6. INIReader (optional - C++)
# ----------------------------------------------------------------------
if(WITH_INIREADER)
    message(STATUS "With INIReader")
    # C++ library
    add_library(INIReader STATIC cpp/INIReader.cpp)
    target_include_directories(INIReader PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/cpp)
    target_compile_options(INIReader PRIVATE ${EXTRA_ARGS})
    target_link_libraries(INIReader PRIVATE inih_interface)

    set_target_properties(INIReader PROPERTIES
            VERSION 0.0.0
            SOVERSION 0
            POSITION_INDEPENDENT_CODE ON
            C_VISIBILITY_PRESET hidden
            CXX_VISIBILITY_PRESET hidden
    )

    # ---- alias‑target ----
    add_library(INIReader_interface INTERFACE)
    add_library(inih::INIReader ALIAS INIReader_interface)

    target_include_directories(INIReader_interface INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/cpp)
    target_link_libraries(INIReader_interface INTERFACE INIReader)
    target_compile_options(INIReader_interface INTERFACE ${EXTRA_ARGS})

    if(DISTRO_INSTALL)
        message(STATUS "Distro install (INIReader) -- TRUE")
        install(TARGETS INIReader
                ARCHIVE DESTINATION lib
                LIBRARY DESTINATION lib
                RUNTIME DESTINATION bin
        )
        install(FILES cpp/INIReader.h DESTINATION include)

        configure_file(pkg-config.in INIReader.pc @ONLY)
        install(FILES ${CMAKE_CURRENT_BINARY_DIR}/INIReader.pc DESTINATION lib/pkgconfig)
    endif()

    add_subdirectory(examples)
endif()
