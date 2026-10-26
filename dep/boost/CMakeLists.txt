# This file is part of the TrinityCore Project. See AUTHORS file for Copyright information
#
# This file is free software; as a special exception the author gives
# unlimited permission to copy and/or distribute it, with or without
# modifications, as long as this notice is preserved.
#
# This program is distributed in the hope that it will be useful, but
# WITHOUT ANY WARRANTY, to the extent permitted by law; without even the
# implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

# 本项目固定使用本地 boost 1.92（与 VS 2026 toolset 14.5 精确匹配）。//By leewheel 20261026
# 无论环境变量 / cache 里 BOOST_ROOT 被设成什么（cmake-gui 老进程里可能还是旧值 1_78_0），
# 只要本地 1.92 存在就一律以它为准，并直接钉死 Boost_DIR —— 这样 vcpkg 工具链的
# find_package 劫持（vcpkg.cmake 的 boost wrapper）也无法把 boost 顶回 vcpkg 版本。
# ⚠️ CMP0144 警告的根因是 find_package 前存在大写 CMake 变量 BOOST_ROOT，vcpkg wrapper
# 内部 push 的策略域里 cmake_policy / CMAKE_POLICY_DEFAULT 都压不住它 ⇒ 正解是
# **彻底 unset(BOOST_ROOT)**，搜索路径改由 BOOST_SEARCH_HINTS / Boost_DIR 提供。
# End By leewheel
set(_LEEWHEEL_BOOST_PIN_ROOT "C:/local/boost_1_92_0")
set(_LEEWHEEL_BOOST_PIN_DIR "${_LEEWHEEL_BOOST_PIN_ROOT}/lib64-msvc-14.5/cmake/Boost-1.92.0")
if(EXISTS "${_LEEWHEEL_BOOST_PIN_DIR}/BoostConfig.cmake")
  #By leewheel 20261026 钉死 Boost_DIR（config 模式直接用它，优先级高于一切搜索路径）
  set(Boost_DIR "${_LEEWHEEL_BOOST_PIN_DIR}" CACHE PATH "Pinned local boost 1.92 config directory" FORCE)
  #End By leewheel
endif()
unset(_LEEWHEEL_BOOST_PIN_DIR)

set(BOOST_SEARCH_HINTS)
if(WIN32)
  #By leewheel 20261026 环境变量可能指向旧版本（如 1_78_0）；本地 1.92 存在时只认它，
  # 并且不落成 BOOST_ROOT 变量（见顶部 CMP0144 说明），直接生成 HINTS
  if(EXISTS "${_LEEWHEEL_BOOST_PIN_ROOT}/lib64-msvc-14.5/cmake")
    set(BOOST_SEARCH_HINTS "${_LEEWHEEL_BOOST_PIN_ROOT}/lib64-msvc-14.5/cmake")
  elseif(DEFINED ENV{BOOST_ROOT})
    set(BOOST_ROOT $ENV{BOOST_ROOT})
  endif()
  #End By leewheel
  if(DEFINED BOOST_ROOT AND MSVC)
    # insert a dot (.) character before last digit of MSVC_TOOLSET_VERSION
    # turn 143 into 14.3
    string(LENGTH "${MSVC_TOOLSET_VERSION}" _BOOST_MSVC_TOOLSET_VERSION_LENGTH)
    math(EXPR _BOOST_MSVC_TOOLSET_VERSION_LENGTH "${_BOOST_MSVC_TOOLSET_VERSION_LENGTH} - 1" OUTPUT_FORMAT DECIMAL)
    string(SUBSTRING "${MSVC_TOOLSET_VERSION}" 0 ${_BOOST_MSVC_TOOLSET_VERSION_LENGTH} _BOOST_MSVC_TOOLSET_VERSION_MAJOR)
    string(SUBSTRING "${MSVC_TOOLSET_VERSION}" ${_BOOST_MSVC_TOOLSET_VERSION_LENGTH} -1 _BOOST_MSVC_TOOLSET_VERSION_MINOR)

    set(_BOOST_CANDIDATE_DIR "${BOOST_ROOT}/lib${PLATFORM}-msvc-${_BOOST_MSVC_TOOLSET_VERSION_MAJOR}.${_BOOST_MSVC_TOOLSET_VERSION_MINOR}/cmake")
    if(EXISTS "${_BOOST_CANDIDATE_DIR}")
      set(BOOST_SEARCH_HINTS "${_BOOST_CANDIDATE_DIR}")
    else()
      # Fallback: find any lib{PLATFORM}-msvc-*/cmake directory (ABI-compatible toolsets)
      file(GLOB _BOOST_FALLBACK_DIRS "${BOOST_ROOT}/lib${PLATFORM}-msvc-*/cmake")
      list(LENGTH _BOOST_FALLBACK_DIRS _BOOST_FALLBACK_COUNT)
      if(_BOOST_FALLBACK_COUNT GREATER 0)
        list(GET _BOOST_FALLBACK_DIRS 0 BOOST_SEARCH_HINTS)
        message(STATUS "Boost: exact toolset ${_BOOST_MSVC_TOOLSET_VERSION_MAJOR}.${_BOOST_MSVC_TOOLSET_VERSION_MINOR} not found, using ${BOOST_SEARCH_HINTS}")
      endif()
      unset(_BOOST_FALLBACK_DIRS)
      unset(_BOOST_FALLBACK_COUNT)
    endif()
    unset(_BOOST_CANDIDATE_DIR)

    unset(_BOOST_MSVC_TOOLSET_VERSION_LENGTH)
    unset(_BOOST_MSVC_TOOLSET_VERSION_MAJOR)
    unset(_BOOST_MSVC_TOOLSET_VERSION_MINOR)
  endif()

  set(Boost_USE_STATIC_LIBS        ON)
  set(Boost_USE_MULTITHREADED      ON)
  set(Boost_USE_STATIC_RUNTIME     OFF)
endif()

if (WIN32)
  # On windows the requirements are higher according to the wiki.
  set(BOOST_REQUIRED_VERSION 1.78)
else()
  set(BOOST_REQUIRED_VERSION 1.74)
endif()

#By leewheel 20261026 find_package 前彻底清掉大写 BOOST_ROOT（含 cache），消除 CMP0144 警告
unset(BOOST_ROOT)
unset(BOOST_ROOT CACHE)
#End By leewheel

find_package(Boost ${BOOST_REQUIRED_VERSION}
  REQUIRED
  COMPONENTS
    system
    filesystem
    thread
    program_options
    iostreams
    regex
    locale
  CONFIG
  HINTS
    ${BOOST_SEARCH_HINTS})

# Find if Boost was compiled in C++03 mode because it requires -DBOOST_NO_CXX11_SCOPED_ENUMS

include (CheckCXXSourceCompiles)

set(CMAKE_REQUIRED_INCLUDES ${Boost_INCLUDE_DIR})
set(CMAKE_REQUIRED_LIBRARIES ${Boost_SYSTEM_LIBRARY} ${Boost_FILESYSTEM_LIBRARY} ${Boost_IOSTREAMS_LIBRARY})
set(CMAKE_REQUIRED_FLAGS "-std=c++11")
unset(boost_filesystem_copy_links_without_NO_SCOPED_ENUM CACHE)
check_cxx_source_compiles("
  #include <boost/filesystem/path.hpp>
  #include <boost/filesystem/operations.hpp>
  int main() { boost::filesystem::copy_file(boost::filesystem::path(), boost::filesystem::path()); }"
boost_filesystem_copy_links_without_NO_SCOPED_ENUM)
unset(CMAKE_REQUIRED_INCLUDES CACHE)
unset(CMAKE_REQUIRED_LIBRARIES CACHE)
unset(CMAKE_REQUIRED_FLAGS CACHE)

if (NOT boost_filesystem_copy_links_without_NO_SCOPED_ENUM)
  set(OPTIONAL_BOOST_NO_SCOPED_ENUMS -DBOOST_NO_CXX11_SCOPED_ENUMS)
endif()

add_library(boost INTERFACE)

target_link_libraries(boost
  INTERFACE
    ${Boost_LIBRARIES})

target_include_directories(boost
  INTERFACE
    ${Boost_INCLUDE_DIRS})

target_compile_definitions(boost
  INTERFACE
    -DBOOST_DATE_TIME_NO_LIB
    -DBOOST_REGEX_NO_LIB
    -DBOOST_CHRONO_NO_LIB
    ${OPTIONAL_BOOST_NO_SCOPED_ENUMS})
