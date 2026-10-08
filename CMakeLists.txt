cmake_minimum_required(VERSION 3.24)

project(trading_system
    VERSION 0.1.0
    LANGUAGES CXX
)

# ---- Global settings -------------------------------------------------------
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# ---- Warnings (applied to every target) ------------------------------------
add_compile_options(-Wall -Wextra -Wpedantic -Werror)

find_package(Threads REQUIRED)

# ---- Sanitizers (Debug builds only) ----------------------------------------
option(TS_ENABLE_SANITIZERS "Build with ASan+UBSan in Debug" ON)
option(TS_ENABLE_TSAN "Build with ThreadSanitizer (mutually exclusive with ASan)" OFF)
if(TS_ENABLE_TSAN)
    message(WARNING "TSan + ASan are mutually exclusive; TSan wins.")
    set(TS_ENABLE_SANITIZERS OFF)
endif()
if(TS_ENABLE_SANITIZERS AND CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_compile_options(-fsanitize=address,undefined -fno-omit-frame-pointer)
    add_link_options(-fsanitize=address,undefined)
elseif(TS_ENABLE_TSAN AND CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_compile_options(-fsanitize=thread -fno-omit-frame-pointer)
    add_link_options(-fsanitize=thread)
endif()

# ---- Main executable -------------------------------------------------------
add_executable(trading_system
    src/main.cpp
)

# ============================================================================
#  FUTURE STEPS — uncomment only when the referenced source files exist.
#  CMake fails at configure time if a listed source is missing.
# ============================================================================

# ---- [Month 1: week 4] hot-path flags for bench/release ---------------------
# set(TS_HOT_PATH_FLAGS -O3 -march=native -flto -fno-exceptions -fno-rtti)

# ---- [Month 1: weeks 1-3] ts_common library ---------------------------------
# add_library(ts_common STATIC
#     src/common/queue/spsc_queue.cpp
#     src/common/allocator/pool_allocator.cpp
#     src/common/allocator/arena_allocator.cpp
#     src/common/log/async_logger.cpp
#     src/common/time/timer.cpp
# )
# target_include_directories(ts_common PUBLIC include)
# target_link_libraries(ts_common PUBLIC Threads::Threads)

# ---- [Month 2: weeks 6-8] remaining component libraries ---------------------
# add_library(ts_feed STATIC
#     src/feed/feed_handler.cpp
#     src/feed/feed_generator.cpp
#     src/feed/protocol.cpp
# )
# target_link_libraries(ts_feed PUBLIC ts_common)
#
# add_library(ts_book STATIC
#     src/book/order_book.cpp
#     src/book/order.cpp
# )
# target_link_libraries(ts_book PUBLIC ts_common)
#
# add_library(ts_engine STATIC
#     src/engine/matching_engine.cpp
#     src/engine/order_state_machine.cpp
#     src/engine/risk.cpp
# )
# target_link_libraries(ts_engine PUBLIC ts_book)
#
# add_library(ts_gateway STATIC
#     src/gateway/order_gateway.cpp
#     src/gateway/order_manager.cpp
# )
# target_link_libraries(ts_gateway PUBLIC ts_common)
#
# add_library(ts_strategy STATIC
#     src/strategy/spread_strategy.cpp
# )
# target_link_libraries(ts_strategy PUBLIC ts_book ts_engine ts_gateway)
#
# target_link_libraries(trading_system PRIVATE
#     ts_common ts_feed ts_book ts_engine ts_gateway ts_strategy
# )

# ---- [Month 1: week 1] doctest + unit tests ---------------------------------
# include(FetchContent)
# FetchContent_Declare(doctest
#     GIT_REPOSITORY https://github.com/doctest/doctest
#     GIT_TAG v2.4.11
# )
# FetchContent_MakeAvailable(doctest)
#
# add_executable(unit_tests
#     tests/test_main.cpp
#     tests/test_spsc.cpp
#     tests/test_pool.cpp
#     tests/test_order_book.cpp
#     tests/test_matching.cpp
#     tests/test_protocol.cpp
# )
# target_link_libraries(unit_tests PRIVATE
#     doctest::doctest ts_common ts_feed ts_book ts_engine ts_gateway
# )
#
# enable_testing()
# add_test(NAME unit_tests COMMAND unit_tests)

# ---- [Month 1: week 4] benchmarks -------------------------------------------
# add_executable(bench
#     bench/bench_harness.cpp
#     bench/bench_spsc.cpp
#     bench/bench_order_book.cpp
# )
# target_compile_options(bench PRIVATE ${TS_HOT_PATH_FLAGS})
# target_link_libraries(bench PRIVATE ts_common ts_book)
