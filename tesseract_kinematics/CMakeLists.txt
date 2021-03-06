cmake_minimum_required(VERSION 3.5.0)

# Extract package name and version
find_package(ros_industrial_cmake_boilerplate REQUIRED)
extract_package_metadata(pkg)
project(${pkg_extracted_name} VERSION ${pkg_extracted_version} LANGUAGES CXX)

if(WIN32)
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()

## System dependencies are found with CMake's conventions
find_package(Eigen3 REQUIRED)
find_package(orocos_kdl REQUIRED)
find_package(console_bridge REQUIRED)
find_package(tesseract_scene_graph REQUIRED)
find_package(tesseract_common REQUIRED)
find_package(opw_kinematics REQUIRED)

if(NOT TARGET console_bridge::console_bridge)
  add_library(console_bridge::console_bridge INTERFACE IMPORTED)
  set_target_properties(console_bridge::console_bridge PROPERTIES INTERFACE_INCLUDE_DIRECTORIES ${console_bridge_INCLUDE_DIRS})
  set_target_properties(console_bridge::console_bridge PROPERTIES INTERFACE_LINK_LIBRARIES ${console_bridge_LIBRARIES})
else()
  get_target_property(CHECK_INCLUDE_DIRECTORIES console_bridge::console_bridge INTERFACE_INCLUDE_DIRECTORIES)
  if (NOT ${CHECK_INCLUDE_DIRECTORIES})
    set_target_properties(console_bridge::console_bridge PROPERTIES INTERFACE_INCLUDE_DIRECTORIES ${console_bridge_INCLUDE_DIRS})
  endif()
endif()

initialize_code_coverage()
set(COVERAGE_EXCLUDE /usr/* /opt/* ${CMAKE_CURRENT_LIST_DIR}/test/* /*/gtest/* /*/bullet/LinearMath/*)
add_code_coverage_all_targets(EXCLUDE ${COVERAGE_EXCLUDE})

# Load variable for clang tidy args, compiler options and cxx version
tesseract_variables()

# Create interface for core
add_library(${PROJECT_NAME}_core
  src/core/rop_inverse_kinematics.cpp
  src/core/rep_inverse_kinematics.cpp)
target_link_libraries(${PROJECT_NAME}_core PUBLIC tesseract::tesseract_scene_graph console_bridge::console_bridge)
target_compile_options(${PROJECT_NAME}_core PRIVATE ${TESSERACT_COMPILE_OPTIONS_PRIVATE})
target_compile_options(${PROJECT_NAME}_core PUBLIC ${TESSERACT_COMPILE_OPTIONS_PUBLIC})
target_compile_definitions(${PROJECT_NAME}_core PUBLIC ${TESSERACT_COMPILE_DEFINITIONS})
target_clang_tidy(${PROJECT_NAME}_core ARGUMENTS ${TESSERACT_CLANG_TIDY_ARGS} ENABLE ${TESSERACT_ENABLE_CLANG_TIDY})
target_cxx_version(${PROJECT_NAME}_core PUBLIC VERSION ${TESSERACT_CXX_VERSION})
target_code_coverage(${PROJECT_NAME}_core ALL EXCLUDE ${COVERAGE_EXCLUDE} ENABLE ${TESSERACT_ENABLE_CODE_COVERAGE})
target_include_directories(${PROJECT_NAME}_core PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
    "$<INSTALL_INTERFACE:include>")
target_include_directories(${PROJECT_NAME}_core SYSTEM PUBLIC
    ${EIGEN3_INCLUDE_DIRS})

# Create ikfast kinematics interface
add_library(${PROJECT_NAME}_ikfast INTERFACE)
target_link_libraries(${PROJECT_NAME}_ikfast INTERFACE ${PROJECT_NAME}_core console_bridge::console_bridge)
target_compile_definitions(${PROJECT_NAME}_ikfast INTERFACE -DIKFAST_NO_MAIN -DIKFAST_CLIBRARY -DIKFAST_HAS_LIBRARY)
target_compile_options(${PROJECT_NAME}_ikfast INTERFACE ${TESSERACT_COMPILE_OPTIONS_PUBLIC})
target_compile_definitions(${PROJECT_NAME}_ikfast INTERFACE ${TESSERACT_COMPILE_DEFINITIONS})
target_clang_tidy(${PROJECT_NAME}_ikfast ARGUMENTS ${TESSERACT_CLANG_TIDY_ARGS} ENABLE ${TESSERACT_ENABLE_CLANG_TIDY})
target_cxx_version(${PROJECT_NAME}_ikfast INTERFACE VERSION ${TESSERACT_CXX_VERSION})
target_code_coverage(${PROJECT_NAME}_ikfast INTERFACE ALL EXCLUDE ${COVERAGE_EXCLUDE} ENABLE ${TESSERACT_ENABLE_CODE_COVERAGE})
target_include_directories(${PROJECT_NAME}_ikfast INTERFACE
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
    "$<INSTALL_INTERFACE:include>")
target_include_directories(${PROJECT_NAME}_ikfast SYSTEM INTERFACE
    ${EIGEN3_INCLUDE_DIRS})

# Add KDL implementation of tesseract_kinematics
add_library(${PROJECT_NAME}_kdl
  src/kdl/kdl_fwd_kin_chain.cpp
  src/kdl/kdl_fwd_kin_tree.cpp
  src/kdl/kdl_inv_kin_chain_lma.cpp
  src/kdl/kdl_inv_kin_chain_nr.cpp
)
target_link_libraries(${PROJECT_NAME}_kdl PUBLIC ${PROJECT_NAME}_core tesseract::tesseract_scene_graph orocos-kdl console_bridge::console_bridge)
target_compile_options(${PROJECT_NAME}_kdl PRIVATE ${TESSERACT_COMPILE_OPTIONS_PRIVATE})
target_compile_options(${PROJECT_NAME}_kdl PUBLIC ${TESSERACT_COMPILE_OPTIONS_PUBLIC})
target_compile_definitions(${PROJECT_NAME}_kdl PUBLIC ${TESSERACT_COMPILE_DEFINITIONS})
target_clang_tidy(${PROJECT_NAME}_kdl ARGUMENTS ${TESSERACT_CLANG_TIDY_ARGS} ENABLE ${TESSERACT_ENABLE_CLANG_TIDY})
target_cxx_version(${PROJECT_NAME}_kdl PUBLIC VERSION ${TESSERACT_CXX_VERSION})
target_code_coverage(${PROJECT_NAME}_kdl ALL EXCLUDE ${COVERAGE_EXCLUDE} ENABLE ${TESSERACT_ENABLE_CODE_COVERAGE})
if(KDL_VERSION VERSION_LESS "1.4.0")
    target_compile_definitions(${PROJECT_NAME}_kdl PUBLIC KDL_LESS_1_4_0=ON)
endif()
target_include_directories(${PROJECT_NAME}_kdl PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
    "$<INSTALL_INTERFACE:include>")
target_include_directories(${PROJECT_NAME}_kdl SYSTEM PUBLIC
    ${EIGEN3_INCLUDE_DIRS}
    ${orocos_kdl_INCLUDE_DIRS}) # Orocos did not export target correctly

# Add OPW implementation of tesseract_kinematics
add_library(${PROJECT_NAME}_opw src/opw/opw_inv_kin.cpp)
target_link_libraries(${PROJECT_NAME}_opw PUBLIC ${PROJECT_NAME}_core tesseract::tesseract_scene_graph opw_kinematics::opw_kinematics console_bridge::console_bridge)
target_compile_options(${PROJECT_NAME}_opw PRIVATE ${TESSERACT_COMPILE_OPTIONS_PRIVATE})
target_compile_options(${PROJECT_NAME}_opw PUBLIC ${TESSERACT_COMPILE_OPTIONS_PUBLIC})
target_compile_definitions(${PROJECT_NAME}_opw PUBLIC ${TESSERACT_COMPILE_DEFINITIONS})
target_clang_tidy(${PROJECT_NAME}_opw ARGUMENTS ${TESSERACT_CLANG_TIDY_ARGS} ENABLE ${TESSERACT_ENABLE_CLANG_TIDY})
target_cxx_version(${PROJECT_NAME}_opw PUBLIC VERSION ${TESSERACT_CXX_VERSION})
target_code_coverage(${PROJECT_NAME}_opw ALL EXCLUDE ${COVERAGE_EXCLUDE} ENABLE ${TESSERACT_ENABLE_CODE_COVERAGE})
target_include_directories(${PROJECT_NAME}_opw PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
    "$<INSTALL_INTERFACE:include>")
target_include_directories(${PROJECT_NAME}_opw SYSTEM PUBLIC
    ${EIGEN3_INCLUDE_DIRS})

configure_package(NAMESPACE tesseract TARGETS ${PROJECT_NAME}_core ${PROJECT_NAME}_kdl ${PROJECT_NAME}_ikfast ${PROJECT_NAME}_opw)

# Mark cpp header files for installation
install(DIRECTORY include/${PROJECT_NAME}
  DESTINATION include
  FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
  PATTERN ".svn" EXCLUDE
)

if (TESSERACT_ENABLE_TESTING)
  enable_testing()
  add_run_tests_target(ENABLE ${TESSERACT_ENABLE_RUN_TESTING})
  add_subdirectory(test)
endif()
