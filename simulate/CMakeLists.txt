cmake_minimum_required(VERSION 3.16)
project(unitree_mujoco)

# Build flags
if (CMAKE_BUILD_TYPE MATCHES "Debug")
  message("Debug mode")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17 -g -O0 -fPIC")
else()
  message("Release mode")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17 -O3 -DNDEBUG -fPIC")
endif()

# --- Dependencies
list(APPEND CMAKE_PREFIX_PATH "/opt/unitree_robotics/lib/cmake")
find_package(unitree_sdk2 REQUIRED)
find_package(Boost REQUIRED COMPONENTS program_options)
find_package(Threads REQUIRED)
find_package(fmt CONFIG REQUIRED)

# mujoco via config or pkg-config
find_package(mujoco QUIET CONFIG)
if (NOT mujoco_FOUND)
  find_package(PkgConfig REQUIRED)
  pkg_check_modules(MUJOCO REQUIRED mujoco)
endif()

# GLFW (target may be glfw or glfw::glfw)
find_package(glfw3 QUIET)
set(GLFW_TARGET glfw)
if (TARGET glfw::glfw)
  set(GLFW_TARGET glfw::glfw)
endif()

# --- Real path to MuJoCo's sample app sources (sibling of unitree_mujoco)
# We are in: /workspace/src/unitree_mujoco/simulate
# MuJoCo app src lives in: /workspace/src/mujoco/simulate
set(MUJOCO_APP_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../../mujoco/simulate")

# --- Sources
file(GLOB SIM_APP_SRC
  "${MUJOCO_APP_DIR}/glfw_*.cc"
  "${MUJOCO_APP_DIR}/platform_*.cc"
  "${MUJOCO_APP_DIR}/simulate.cc"
)

add_executable(unitree_mujoco
  ${SIM_APP_SRC}
  src/joystick/joystick.cc
  src/lodepng/lodepng.cpp
  src/main.cc
)

add_executable(jstest
  src/joystick/jstest.cc
  src/joystick/joystick.cc
)

# --- Includes (per-target)
target_include_directories(unitree_mujoco PRIVATE
  "${CMAKE_CURRENT_SOURCE_DIR}/mujoco/include"   # your local headers, if any
  "${MUJOCO_APP_DIR}"                            # <-- contains glfw_adapter.h etc.
  "${CMAKE_CURRENT_SOURCE_DIR}/src/lodepng"
)

# Link (prefer imported mujoco target if present)
if (TARGET mujoco::mujoco)
  target_link_libraries(unitree_mujoco PRIVATE
    mujoco::mujoco
    ${GLFW_TARGET}
    yaml-cpp
    unitree_sdk2
    Boost::program_options
    fmt::fmt
    Threads::Threads
  )
else()
  target_include_directories(unitree_mujoco PRIVATE ${MUJOCO_INCLUDE_DIRS})
  target_link_directories(unitree_mujoco PRIVATE ${MUJOCO_LIBRARY_DIRS})
  target_link_libraries(unitree_mujoco PRIVATE
    ${MUJOCO_LIBRARIES}
    ${GLFW_TARGET}
    yaml-cpp
    unitree_sdk2
    Boost::program_options
    fmt::fmt
    Threads::Threads
  )
endif()

# jstest should NOT depend on mujoco
target_link_libraries(jstest PRIVATE Threads::Threads)
