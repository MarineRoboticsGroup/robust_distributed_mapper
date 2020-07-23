cmake_minimum_required(VERSION 2.8 FATAL_ERROR)
project(distributed-mapper)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -fPIC")
add_definitions(-DLOG_DIR="${PROJECT_SOURCE_DIR}/..")

###########################################################################
# Find GTSAM
find_package(GTSAM REQUIRED )
include_directories(${GTSAM_INCLUDE_DIR})
link_directories(${GTSAM_LIBRARY_DIRS})
message("GTSAM Libraries: ${GTSAM_LIBRARIES}")

# for unittest examples

###########################################################################
# Find Boost
find_package(Boost COMPONENTS serialization filesystem system program_options thread REQUIRED)
include_directories(${Boost_INCLUDE_DIRS})
link_libraries(${Boost_LIBRARIES})
message("Boost Libraries: ${Boost_LIBRARIES}")

###########################################################################
# Pairwise Consistency Maximization

# Overall include directories
include_directories(
  include
  include/pairwise_consistency_maximization
  ${CMAKE_CURRENT_SOURCE_DIR}/src/pairwise_consistency_maximization/third_parties/fast_max-clique_finder/src/
)

# Graph utils library
add_library(graph_utils
        src/pairwise_consistency_maximization/graph_utils/graph_utils_functions.cpp
)
target_link_libraries(graph_utils
        gtsam
)

# Robot local map library
add_library(robot_measurements
        src/pairwise_consistency_maximization/robot_measurements/robot_measurements.cpp
        src/pairwise_consistency_maximization/robot_measurements/interrobot_measurements.cpp
        src/pairwise_consistency_maximization/robot_measurements/robot_local_map.cpp
        )
target_link_libraries(robot_measurements
        graph_utils
        )

# Paiwise Consistency computation library
add_library(pairwise_consistency
   src/pairwise_consistency_maximization/pairwise_consistency/pairwise_consistency.cpp
)
target_link_libraries(pairwise_consistency
       robot_measurements
)

# Maximum Clique Solver
add_library(fast_max-clique_finder
    src/pairwise_consistency_maximization/third_parties/fast_max-clique_finder/src/findClique.h
    src/pairwise_consistency_maximization/third_parties/fast_max-clique_finder/src/graphIO.h
    src/pairwise_consistency_maximization/third_parties/fast_max-clique_finder/src/findClique.cpp
    src/pairwise_consistency_maximization/third_parties/fast_max-clique_finder/src/findCliqueHeu.cpp 
    src/pairwise_consistency_maximization/third_parties/fast_max-clique_finder/src/utils.cpp 
    src/pairwise_consistency_maximization/third_parties/fast_max-clique_finder/src/graphIO.cpp
)
target_compile_options(fast_max-clique_finder PRIVATE -w)

# Global map solver library
add_library(global_map_solver
   src/pairwise_consistency_maximization/global_map/global_map.cpp
)
target_link_libraries(global_map_solver
   graph_utils
   pairwise_consistency
   fast_max-clique_finder
)

# Distributed PCM library
add_library(distributed_pcm
        src/pairwise_consistency_maximization/distributed_pcm/distributed_pcm.cpp
)
target_link_libraries(distributed_pcm
        global_map_solver
)

# Distributed Mapper
add_library (distributed_mapper
        include/distributed_mapper/run_distributed_mapper.h
        src/distributed_mapper/distributed_mapper.cpp
        src/distributed_mapper/evaluation_utils.cpp
        include/distributed_mapper/distributed_mapper_utils.h
        src/distributed_mapper/distributed_mapper_utils.cpp
        src/distributed_mapper/run_distributed_mapper.cpp
        )

target_link_libraries(distributed_mapper
        ${Boost_LIBRARIES}
        gtsam
        distributed_pcm)

###########################################################################
# Add example executable
add_executable (robust_distributed_optimization_example_2robots examples/robust_distributed_optimization_example_2robots.cpp)
target_link_libraries (robust_distributed_optimization_example_2robots
        distributed_mapper)

###########################################################################
# Tests
# add_subdirectory(tests)

###########################################################################
# Install
install(TARGETS graph_utils DESTINATION lib)
file(GLOB headers_graph_utils "include/pairwise_consistency_maximization/graph_utils/*.h")
install(FILES ${headers_graph_utils} DESTINATION include/graph_utils/)

install(TARGETS robot_measurements DESTINATION lib)
file(GLOB headers_robot_measurements "include/pairwise_consistency_maximization/robot_measurements/*.h")
install(FILES ${headers_robot_measurements} DESTINATION include/robot_measurements/)

install(TARGETS pairwise_consistency DESTINATION lib)
file(GLOB headers_pairwise_consistency "include/pairwise_consistency_maximization/pairwise_consistency/*.h")
install(FILES ${headers_pairwise_consistency} DESTINATION include/pairwise_consistency/)

install(TARGETS fast_max-clique_finder DESTINATION lib)
file(GLOB headers_fmc "${CMAKE_CURRENT_SOURCE_DIR}/src/pairwise_consistency_maximization/third_parties/fast_max-clique_finder/src/*.h")
install(FILES ${headers_fmc} DESTINATION include/distributed_mapper/)

install(TARGETS global_map_solver DESTINATION lib)
file(GLOB headers_global_map "include/pairwise_consistency_maximization/global_map/*.h")
install(FILES ${headers_global_map} DESTINATION include/global_map/)

install(TARGETS distributed_pcm DESTINATION lib)
file(GLOB headers_dpcm "include/pairwise_consistency_maximization/distributed_pcm/*.h")
install(FILES ${headers_dpcm} DESTINATION include/distributed_pcm/)

install(TARGETS distributed_mapper DESTINATION lib)
file(GLOB headers_distributed_mapper "include/distributed_mapper/*.h")
install(FILES ${headers_distributed_mapper} DESTINATION include/distributed_mapper/)
