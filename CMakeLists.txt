cmake_minimum_required(VERSION 3.8)
project(metav_detection)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# Add -pg to CMAKE_CXX_FLAGS_DEBUG to enable profiling
# set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pg")

find_package(ament_cmake_auto REQUIRED)
ament_auto_find_build_dependencies()

# importing CUDA
set(CUDA_MIN_VERSION "12.0")
find_package(CUDA REQUIRED)
if( ${CUDA_VERSION} GREATER_EQUAL 11.8)
  set(CMAKE_CUDA_ARCHITECTURES "89;87;86;80;75;72;70;61;60")
else()
  set(CMAKE_CUDA_ARCHITECTURES "87;86;80;75;72;70;61;60")
endif()

# metav_detection
ament_auto_add_library(${PROJECT_NAME} SHARED
  src/metav_detector.cpp
  src/yolo_engine.cpp
  src/pnp_solver.cpp
  src/camera.cpp
  src/mv_camera.cpp
)
ament_target_dependencies(${PROJECT_NAME}
  rclcpp
  rclcpp_components
  sensor_msgs
  OpenCV
  cv_bridge
  image_transport
  metav_interfaces
  tf2_geometry_msgs
  tf2
  camera_info_manager
  visualization_msgs
)
if (CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64")
  target_link_directories(metav_detection PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/mvsdk/lib/arm64
  )
elseif (CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64")
  target_link_directories(metav_detection PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/mvsdk/lib/amd64
  )
else ()
  message(FATAL_ERROR "Unsupported platform")
endif()
target_link_libraries(${PROJECT_NAME}
  nvinfer
  nvonnxparser
  nvinfer_plugin
  ${CUDA_LIBRARIES}
  ${CUDA_npp_LIBRARY}
  ${CUDA_nppc_LIBRARY}
  ${CUDA_nppi_LIBRARY}
  MVSDK
  fmt
)
target_include_directories(${PROJECT_NAME} PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
  ${CUDA_INCLUDE_DIRS}
  ${CMAKE_CURRENT_SOURCE_DIR}/mvsdk/include
)
rclcpp_components_register_node(${PROJECT_NAME} 
  PLUGIN "metav_detection::IrmDetector"
  EXECUTABLE metav_detector
)

if(BUILD_TESTING)
  # find_package(ament_lint_auto REQUIRED)
  # the following line skips the linter which checks for copyrights
  # comment the line when a copyright and license is added to all source files
  # set(ament_cmake_copyright_FOUND TRUE)
  # the following line skips cpplint (only works in a git repo)
  # comment the line when this package is in a git repo and when
  # a copyright and license is added to all source files
  # set(ament_cmake_cpplint_FOUND TRUE)
  # ament_lint_auto_find_test_dependencies()

  find_package(ament_cmake_gtest REQUIRED)
  ament_add_gtest(metav_detection_yolo_test
    test/yolo_test.cpp
  )
  target_link_libraries(metav_detection_yolo_test
    metav_detection
  )
  ament_add_gtest(metav_detection_camera_test
    test/camera_test.cpp
  )
  target_link_libraries(metav_detection_camera_test
    metav_detection
  )
  ament_add_gtest(metav_detection_triple_buffer_test
    test/triple_buffer_test.cpp
  )
  target_link_libraries(metav_detection_triple_buffer_test
    metav_detection
  )
endif()

ament_auto_package(
  INSTALL_TO_SHARE
  models
  config
  test
)
