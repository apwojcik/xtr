cmake_minimum_required(VERSION 3.15)
project(xtr VERSION 3.0.0 LANGUAGES C CXX)

option(BUILD_BENCHMARK "Build benchmark test" ON)
option(BUILD_SINGLE_HEADER "Build header-only" ON)
option(INSTALL_DOCS "Install man pages" ON)
option(ENABLE_EXCEPTIONS "Enable the use of exceptions" ON)
option(ENABLE_LTO "Enable link-time optimisation" ON)
option(XTR_INSTALL "Enable install for the binaries" ON)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

find_package(Threads REQUIRED)
find_package(fmt REQUIRED)

if (CMAKE_CXX_COMPILER_ID MATCHES ".*Clang" OR CMAKE_CXX_COMPILER_ID MATCHES ".*GNU")
    add_compile_options(-Wall -Wextra -Wconversion -Wshadow -Wcast-qual -Wformat=2 -pedantic -pipe)
endif()

if (NOT ENABLE_EXCEPTIONS)
    add_compile_options(-fno-exceptions)
endif()

if (ENABLE_LTO)
    # CMake's INTERPROCEDURAL_OPTIMIZATION feature isn't used as it does not
    # use gcc-ranlib and gcc-ar (as recommended by
    # https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html).
    add_compile_options(-flto)
    add_link_options(-flto)
endif()

file(GLOB_RECURSE HEADER_FILES include/*.hpp)

add_library(xtr src/buffer.cpp
                src/command_dispatcher.cpp
                src/command_path.cpp
                src/consumer.cpp
                src/fd_storage_base.cpp
                src/fd_storage.cpp
                src/file_descriptor.cpp
                src/io_uring_fd_storage.cpp
                src/logger.cpp
                src/log_level.cpp
                src/matcher.cpp
                src/memory_mapping.cpp
                src/mirrored_memory_mapping.cpp
                src/pagesize.cpp
                src/posix_fd_storage.cpp
                src/regex_matcher.cpp
                src/sink.cpp
                src/throw.cpp
                src/tsc.cpp
                src/wildcard_matcher.cpp)
target_link_libraries(xtr fmt::fmt Threads::Threads)
target_include_directories(xtr PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>)
target_compile_definitions(xtr PRIVATE XTR_FUNC=)

add_executable(xtrctl src/xtrctl.cpp)
target_link_libraries(xtrctl xtr)

if(XTR_INSTALL)
    include(GNUInstallDirs)
    install(TARGETS xtr xtrctl
            EXPORT xtrTargets
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
    export(EXPORT xtrTargets
            FILE ${CMAKE_CURRENT_BINARY_DIR}/xtr-targets.cmake
            NAMESPACE xtr::)
    install(EXPORT xtrTargets
            FILE xtr-targets.cmake
            NAMESPACE xtr::
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/xtr)
    install(DIRECTORY ${CMAKE_SOURCE_DIR}/include/
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
    include(CMakePackageConfigHelpers)
    configure_package_config_file(
            xtr-config.cmake.in
            ${CMAKE_CURRENT_BINARY_DIR}/xtr-config.cmake
            INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/xtr)
    write_basic_package_version_file(
            ${CMAKE_CURRENT_BINARY_DIR}/xtr-config-version.cmake
            VERSION ${PROJECT_VERSION}
            COMPATIBILITY AnyNewerVersion)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/xtr-config.cmake
                  ${CMAKE_CURRENT_BINARY_DIR}/xtr-config-version.cmake
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/xtr)
endif()

if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME AND BUILD_TESTING)
    include(CTest)
    enable_testing()
    add_subdirectory(test)
endif()

if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME AND BUILD_BENCHMARK)
    add_subdirectory(benchmark)
endif()

if(BUILD_SINGLE_HEADER)
    execute_process(COMMAND bash -c scripts/make_single_include.sh
                    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
    install(FILES single_include/xtr/logger.hpp
            DESTINATION single_include/xtr)
endif()

if(INSTALL_DOCS)
    install(FILES docs/xtrctl.1
            DESTINATION man/man1)
    install(FILES docs/libxtr.3 docs/libxtr-quickstart.3 docs/libxtr-userguide.3
            DESTINATION man/man3)
endif()
