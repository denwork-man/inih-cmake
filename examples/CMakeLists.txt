# -------------------------------------------------
#  examples/CMakeLists.txt
#  examples/meson.build conversion → CMake
# -------------------------------------------------

# 1. The path to the supporting script runtime, from tests.
#    In the meson file, it's retrieved from the project roo:
#        tests/runtest.sh
#    We do the same in CMake.
find_program(RUNTEST_EXE NAMES runtest.sh
        PATHS ${CMAKE_SOURCE_DIR}/tests
        NO_DEFAULT_PATH)

if (NOT RUNTEST_EXE)
    message(STATUS "runtest.sh not found - example tests will be skipped")
    return()
endif()

# 2. Sources that each examples needs.
#    * src_inih - main parser (ini.c)
#    * src_INIReader - C++ wrapper (cpp/INIReader.cpp)
#    Both are declared in the root CMakeLists as target
#    `inih` and `INIReader`. We will simply refer to them.
#    If necessary, you can add them directly
#    but using already built targets makes it easier
#    to maintain dependencies.

# 3. Examples' description (corresponds to info from `tests` in meson.build)
set(EXAMPLE_LIST
        # name              additional_source        additiona_options
        "INIReaderExample"  "INIReaderExample.cpp"   ""
)

# ------------------------------------------------------------------
# Helping func, create exe-test and register it in CTest
# ------------------------------------------------------------------
function(add_inih_example ex_name src_cpp extra_args)
    # 4.1. Create list of compile flags
    set(compile_opts -Wall)               # warnings always on -- -Wall
    if (extra_args)
        separate_arguments(extra_args_list UNIX_COMMAND "${extra_args}")
        list(APPEND compile_opts ${extra_args_list})
    endif()

    # 4.2. Create executable.
    #      It builds:
    #        * library `inih` (ini.c) - target already exists `inih`
    #        * library `INIReader` (cpp/INIReader.cpp) - target `INIReader`
    #        * example source (src_cpp)
    add_executable(example_${ex_name}
            ${src_cpp}
    )
    #  link builded libraries
    target_link_libraries(example_${ex_name}
            PRIVATE inih          # C-lib
            PRIVATE INIReader     # C++-wrapper
    )
    target_include_directories(example_${ex_name}
            PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/..
    )
    target_compile_options(example_${ex_name} PRIVATE ${compile_opts})

    # 4.3. Adding a test that runs runtest.sh
    #      There are two arguments:
    #        1) cpptest.txt - the reference file
    #        2) executable path
    add_test(NAME example_${ex_name}
            COMMAND ${RUNTEST_EXE}
            ${CMAKE_CURRENT_SOURCE_DIR}/cpptest.txt
            $<TARGET_FILE:example_${ex_name}>
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )
endfunction()

# ------------------------------------------------------------------
# Iteration through all examples from `EXAMPLE_LIST` and invoking `add_inih_example` func
# ------------------------------------------------------------------
list(LENGTH EXAMPLE_LIST n_items)
math(EXPR n_examples "${n_items} / 3")   # every record consists of 3 fields

foreach(i RANGE 0 ${n_examples-1})
    math(EXPR idx_name    "${i} * 3")
    math(EXPR idx_src     "${i} * 3 + 1")
    math(EXPR idx_opts    "${i} * 3 + 2")

    list(GET EXAMPLE_LIST ${idx_name} cur_name)
    list(GET EXAMPLE_LIST ${idx_src}  cur_src)
    list(GET EXAMPLE_LIST ${idx_opts} cur_opts)

    add_inih_example(${cur_name} ${CMAKE_CURRENT_SOURCE_DIR}/${cur_src} "${cur_opts}")
endforeach()

message(STATUS "Examples added")
