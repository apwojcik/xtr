find_package(benchmark QUIET)

if (CMAKE_SYSTEM MATCHES "Linux")
    find_package(liburing QUIET)
    if (NOT liburing_FOUND)
        message(WARNING "libruing not found")
    endif()
endif()

add_executable(xtr_benchmark logger.cpp main.cpp)

set_target_properties(xtr_benchmark PROPERTIES CXX_STANDARD 20)
target_link_libraries(xtr_benchmark xtr benchmark::benchmark)
if (CMAKE_SYSTEM MATCHES "Linux")
    target_link_libraries(xtr_benchmark xtr liburing::liburing)
endif()
