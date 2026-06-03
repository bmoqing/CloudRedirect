cmake_minimum_required(VERSION 3.20)
project(CloudRedirect LANGUAGES C CXX)

if(NOT WIN32)
    message(FATAL_ERROR "This cleaned custom fork only keeps the Windows build path. Use the upstream project for Linux or Flatpak builds.")
endif()

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

file(READ "${CMAKE_SOURCE_DIR}/Version.props" VERSION_PROPS_CONTENT)
string(REGEX MATCH "<ReleaseVersion>([^<]+)</ReleaseVersion>" _ "${VERSION_PROPS_CONTENT}")
set(CR_RELEASE_VERSION "${CMAKE_MATCH_1}")

if(NOT CR_RELEASE_VERSION)
    set(CR_RELEASE_VERSION "0.0.0")
endif()

execute_process(
    COMMAND git rev-parse --short=7 HEAD
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_SHA
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET
    RESULT_VARIABLE GIT_RESULT
)
if(NOT GIT_RESULT EQUAL 0)
    set(GIT_SHA "unknown")
endif()

execute_process(
    COMMAND git status --porcelain
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_STATUS
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET
)
if(GIT_STATUS)
    set(GIT_SHA "${GIT_SHA}-dirty")
endif()

set(CR_SO_VERSION "${CR_RELEASE_VERSION}+${GIT_SHA}")

set(COMMON_SOURCES
    src/common/protobuf.cpp
    src/common/json.cpp
    src/common/vdf.cpp
    src/common/bkv_stats.cpp
    src/common/remotecache_repair.cpp
    src/common/manifest_store.cpp
    src/common/app_state.cpp
    src/common/token_store.cpp
    src/common/batch_tracker.cpp
    src/common/cloud_staging.cpp
    src/common/pending_ops_journal.cpp
    src/common/rpc_handlers.cpp
    src/common/cloud_work_queue.cpp
    src/common/cloud_storage.cpp
    src/common/local_storage.cpp
    src/common/local_metadata_store.cpp
    src/common/legacy_metadata_cleanup.cpp
    src/common/cli.cpp
    src/common/cloud_provider_base.cpp
    src/common/autocloud_scan.cpp
    src/common/autocloud_bootstrap.cpp
    src/common/steam_kv_injector.cpp
    src/common/parental_bypass.cpp
    src/common/miniz.c
    src/common/miniz_tdef.c
    src/common/miniz_tinfl.c
    src/common/miniz_zip.c
)

set(PROVIDER_SOURCES
    src/providers/google_drive.cpp
    src/providers/onedrive.cpp
    src/providers/webdav_provider.cpp
    src/providers/local_disk_provider.cpp
)

set(PLATFORM_SOURCES
    src/platform/win/dllmain.cpp
    src/platform/win/cloud_intercept.cpp
    src/platform/win/log.cpp
    src/platform/win/http_server.cpp
    src/platform/win/http_util.cpp
    src/platform/win/http_transport_win.cpp
    src/platform/win/token_store_win.cpp
    src/platform/win/platform_win.cpp
)

add_library(cloud_redirect SHARED
    ${COMMON_SOURCES}
    ${PLATFORM_SOURCES}
    ${PROVIDER_SOURCES}
)

set_target_properties(cloud_redirect PROPERTIES PREFIX "")

target_include_directories(cloud_redirect PRIVATE
    src/common
    src/providers
    src/platform/win
)

target_compile_definitions(cloud_redirect PRIVATE
    MINIZ_NO_STDIO
    CR_VERSION_STRING="${CR_SO_VERSION}"
    CR_RELEASE_VERSION="${CR_RELEASE_VERSION}"
    CR_CUSTOM_WEB_DAV_BUILD=1
)

target_link_libraries(cloud_redirect PRIVATE
    Shlwapi Advapi32 ws2_32 winhttp Crypt32 Shell32 Ole32 Psapi
)

add_executable(cloud_redirect_cli
    src/platform/win/cli_main.cpp
)
target_link_libraries(cloud_redirect_cli PRIVATE Shell32 Ole32)
add_dependencies(cloud_redirect_cli cloud_redirect)

add_custom_target(ui ALL
    COMMAND dotnet publish -c Release -r win-x64 --self-contained false -o bin/publish
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/ui
    COMMENT "Publishing CloudRedirect UI"
    DEPENDS cloud_redirect cloud_redirect_cli
)
