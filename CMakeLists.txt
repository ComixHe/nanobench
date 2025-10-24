cmake_minimum_required(VERSION 3.11.4)

project(
  nanobench
  VERSION 4.3.11
  DESCRIPTION "Simple, fast, accurate single-header microbenchmarking functionality for C++11/14/17/20"
  HOMEPAGE_URL "https://github.com/martinus/nanobench"
  LANGUAGES CXX)

# determine whether this is a standalone project or included by other projects
set(NANOBENCH_STANDALONE_PROJECT OFF)

# Modified from https://www.scivision.dev/cmake-project-is-top-level/
if(CMAKE_VERSION VERSION_LESS 3.21)
    get_property(
      not_top
      DIRECTORY
      PROPERTY PARENT_DIRECTORY
    )

    if (NOT not_top)
        set(${PROJECT_NAME}_IS_TOP_LEVEL FALSE)
    else()
        set(${PROJECT_NAME}_IS_TOP_LEVEL TRUE)
    endif()
endif()

if (${PROJECT_NAME}_IS_TOP_LEVEL)
    set(NANOBENCH_STANDALONE_PROJECT ON)
endif()

add_library(${PROJECT_NAME} INTERFACE)
add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

if (NANOBENCH_STANDALONE_PROJECT)
    set(CMAKE_EXPORT_COMPILE_COMMANDS ON) # generate compile_commands.json

    set(NB_cxx_standard "11" CACHE STRING "C++ standard, e.g. 11, 14, 17")

    set(CMAKE_CXX_STANDARD ${NB_cxx_standard})
    set(CMAKE_CXX_STANDARD_REQUIRED ON)
    set(CMAKE_CXX_EXTENSIONS OFF)

    # configuration see .clang-tidy
    if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
        find_program(CLANG_TIDY_PROGRAM NAMES clang-tidy)
        if(CLANG_TIDY_PROGRAM)
            set(CMAKE_CXX_CLANG_TIDY "${CLANG_TIDY_PROGRAM}")
        endif()
    endif()

    find_program(CCACHE_PROGRAM NAMES ccache)
    if(CCACHE_PROGRAM)
        set(CMAKE_CXX_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
    endif()

    add_executable(nb)

    if (NB_sanitizer)
        # Compile with -g and -fno-omit-frame-pointer to get proper debug information in your binary
        set(COMMON_COMPILE_OPTIONS "-g" "-O2" "-fno-omit-frame-pointer")
        set(COMMON_ASAN_COMPILE_OPTIONS "-fsanitize=address,undefined,float-divide-by-zero")
        set(COMMON_ASAN_LINK_OPTIONS ${COMMON_ASAN_COMPILE_OPTIONS})

        if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
            # see https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html#silencing-unsigned-integer-overflow

            target_compile_options(nb PRIVATE ${COMMON_COMPILE_OPTIONS} ${COMMON_ASAN_COMPILE_OPTIONS})
            target_link_options(nb PRIVATE ${COMMON_ASAN_LINK_OPTIONS})

            target_compile_options(nb PRIVATE -fsanitize=integer)
            target_link_libraries(nb PRIVATE -fsanitize=integer)

            target_compile_options(nb PRIVATE -fsanitize=nullability)
            target_link_libraries(nb PRIVATE -fsanitize=nullability)

            # we have to globally set the property here, so it actually works https://cmake.org/pipermail/cmake/2010-March/036020.html
            set_source_files_properties(
                src/test/tutorial_fast_v1.cpp
                src/test/tutorial_fast_v2.cpp
                PROPERTIES COMPILE_FLAGS "-fno-sanitize=integer")
        elseif (CMAKE_CXX_COMPILER_ID MATCHES "GNU")
            # need to use gold linker, otherwise travis gets '/usr/bin/ld: --push-state: unknown option' error
            target_link_libraries(nb PRIVATE -fuse-ld=gold)

            target_compile_options(nb PRIVATE ${COMMON_COMPILE_OPTIONS} ${COMMON_ASAN_COMPILE_OPTIONS})
            target_link_options(nb PRIVATE ${COMMON_ASAN_LINK_OPTIONS})

            target_compile_options(nb PRIVATE -fsanitize=float-cast-overflow)
            target_link_libraries(nb PRIVATE -fsanitize=float-cast-overflow)

            target_compile_options(nb PRIVATE -fsanitize=pointer-compare,pointer-subtract)
            target_link_libraries(nb PRIVATE -fsanitize=pointer-compare,pointer-subtract)
        endif()
    endif()

    add_subdirectory(src)
    add_compile_flags_target(nb)

    target_sources_local(nb PUBLIC .clang-tidy)

    include(GNUInstallDirs)
    set(INSTALL_HEADERS_DIR ${CMAKE_INSTALL_INCLUDEDIR})
    set(INSTALL_CONFIGDIR ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME})

    target_include_directories(${PROJECT_NAME}
      INTERFACE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/include>
        $<INSTALL_INTERFACE:${INSTALL_HEADERS_DIR}>
    )

    # Install headers
    install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/src/include/nanobench.h DESTINATION ${INSTALL_HEADERS_DIR})

    # Install targets file
    install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}Targets)
    install(EXPORT ${PROJECT_NAME}Targets
      FILE
        ${PROJECT_NAME}Targets.cmake
      NAMESPACE
        ${PROJECT_NAME}::
      DESTINATION
        ${INSTALL_CONFIGDIR}
    )

    # Install ${PROJECT_NAME}Config.cmake and ${PROJECT_NAME}ConfigVersion.cmake
    include(CMakePackageConfigHelpers)

    write_basic_package_version_file(
      "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
      VERSION ${PROJECT_VERSION}
      COMPATIBILITY AnyNewerVersion
    )

    configure_package_config_file(
      ${CMAKE_CURRENT_SOURCE_DIR}/cmake/${PROJECT_NAME}Config.cmake.in
      ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
      INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
    )

    install(FILES
      ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
      ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
      DESTINATION ${INSTALL_CONFIGDIR}
    )
else()
    set_target_properties(${PROJECT_NAME} PROPERTIES CXX_STANDARD 17)
    target_include_directories(${PROJECT_NAME} INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/src/include)
endif()
