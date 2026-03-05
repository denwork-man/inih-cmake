# -------------------------------------------------
#  tests/CMakeLists.txt
#  tests/meson.build conversion to CMake by gpt and manual correction
# -------------------------------------------------

# 1. The path to the supporting script runtime.
#    Meson searches for it as `runtest.sh`.
#    If the script is missing, the tests won't be run..
find_program(RUNTEST_EXE NAMES runtest.sh PATHS ${CMAKE_CURRENT_SOURCE_DIR}
        NO_DEFAULT_PATH)

if (NOT RUNTEST_EXE)
    message(STATUS "runtest.sh not found – test suite will be skipped")
    return()                     # nothing to do
endif()

# 2. The sources of the `inih` library, which are required for each test.
#    Files have already been declared as the `inih` target
#    so we simply list it as dependency.
set(INIH_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/../ini.c)   # path to lib sources

# 3. All tests' description (correspond to the original dict `tests` in meson.build)
set(TEST_DEFINES
        # name                        extra_source          extra_defines
        "multi"                       ""                    ""
        "multi_max_line"              ""                    "-DINI_MAX_LINE=20"
        "single"                      ""                    "-DINI_ALLOW_MULTILINE=0"
        "disallow_inline_comments"    ""                    "-DINI_ALLOW_INLINE_COMMENTS=0"
        "stop_on_first_error"         ""                    "-DINI_STOP_ON_FIRST_ERROR=1"
        "handler_lineno"              ""                    "-DINI_HANDLER_LINENO=1"
        "string"                      "unittest_string.c"   "-DINI_MAX_LINE=20"
        "heap"                        ""                    "-DINI_USE_STACK=0"
        "heap_max_line"               ""                    "-DINI_USE_STACK=0;-DINI_MAX_LINE=20;-DINI_INITIAL_ALLOC=20"
        "heap_realloc"                ""                    "-DINI_USE_STACK=0;-DINI_ALLOW_REALLOC=1;-DINI_INITIAL_ALLOC=5"
        "heap_realloc_max_line"       ""                    "-DINI_USE_STACK=0;-DINI_MAX_LINE=20;-DINI_ALLOW_REALLOC=1;-DINI_INITIAL_ALLOC=5"
        "heap_string"                 "unittest_string.c"   "-DINI_USE_STACK=0;-DINI_MAX_LINE=20;-DINI_INITIAL_ALLOC=20"
        "call_handler_on_new_section" ""                    "-DINI_CALL_HANDLER_ON_NEW_SECTION=1"
        "allow_no_value"              ""                    "-DINI_ALLOW_NO_VALUE=1"
        "alloc"                       "unittest_alloc.c"    "-DINI_CUSTOM_ALLOCATOR=1;-DINI_USE_STACK=0;-DINI_ALLOW_REALLOC=1;-DINI_INITIAL_ALLOC=12"
)

# 4. Helping func, create exe-test and register it in CTest
function(add_inih_test test_name src_file extra_defines)
    # 4.1. Creating list of preprocessor flags
    set(defines_list -Wall)               # waringns are always on -- -Wall
    if (extra_defines)
        separate_arguments(extra_defines_list UNIX_COMMAND "${extra_defines}")
        list(APPEND defines_list ${extra_defines_list})
    endif()

    # 4.2. Create executable file
    add_executable(unittest_${test_name}
            ${INIH_SOURCES}
            ${CMAKE_CURRENT_SOURCE_DIR}/${src_file}
    )
    target_include_directories(unittest_${test_name} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/..)
    target_compile_options(unittest_${test_name} PRIVATE ${defines_list})

    # 4.3. Adding test to CTest, by call of runtest.sh
    #     There are two arguments (from meson-file):
    #       1) baseline-file
    #       2) executable path
    add_test(NAME test_${test_name}
            COMMAND ${RUNTEST_EXE}
            ${CMAKE_CURRENT_SOURCE_DIR}/baseline_${test_name}.txt
            $<TARGET_FILE:unittest_${test_name}>
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )
endfunction()

# 5. Iteration through all records from `TEST_DEFINES` and invoking `add_inih_test` func
list(LENGTH TEST_DEFINES n_items)
math(EXPR n_tests "${n_items} / 3")   # every record consists of 3 values

foreach(i RANGE 0 ${n_tests-1})
    math(EXPR idx_name    "${i} * 3")
    math(EXPR idx_source  "${i} * 3 + 1")
    math(EXPR idx_defines "${i} * 3 + 2")

    list(GET TEST_DEFINES ${idx_name}    cur_name)
    list(GET TEST_DEFINES ${idx_source} cur_src)
    list(GET TEST_DEFINES ${idx_defines} cur_defs)

    # If there is no source-file - get default `unittest.c`
    if (cur_src STREQUAL "")
        set(cur_src "unittest.c")
    endif()

    add_inih_test(${cur_name} ${cur_src} "${cur_defs}")
endforeach()

message(STATUS "Tests added")