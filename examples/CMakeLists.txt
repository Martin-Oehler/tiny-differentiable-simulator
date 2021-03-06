#
# Copyright 2020 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.


include_directories(. ..)

set(DL ${CMAKE_DL_LIBS})


find_package(ct_core)
find_package(ct_optcon)
find_package(Ceres)
find_package(PkgConfig)

add_definitions(-DBT_USE_DOUBLE_PRECISION)

#find_package(tinyxml2)
if (tinyxml2_FOUND)
    message("USING system tinyxml2")
    include_directories(${tinyxml2_INCLUDE_DIRS})
else (tinyxml2_FOUND)
    message ("USING third_party/tinyxml2")

    set(tinyxml2_FOUND TRUE)
    set(tinyxml2_INCLUDE_DIRS ../third_party/tinyxml2/include)
    SUBDIRS(../third_party/tinyxml2)
endif (tinyxml2_FOUND)


find_package(crossguid QUIET)
IF (crossguid_FOUND)
    message("USING system crossguid")
ELSE()
    message("USING third_party/crossguid")
    SUBDIRS(../third_party/crossguid)
    set(crossguid_INCLUDE_DIRS ../third_party/crossguid)
    include_directories(${crossguid_INCLUDE_DIRS})
ENDIF()

#find_package(zeromq)
#find_package(cppzmq)

IF(cppzmq_FOUND)
    message("USING system zmq")
ELSE()
    message("USING third_party/zmq")
    SET(cppzmq_FOUND TRUE)
    SUBDIRS(../third_party/zeromq)
    set(cppzmq_INCLUDE_DIRS ../third_party/zeromq/include )
    include_directories(${cppzmq_INCLUDE_DIRS})
    add_definitions(-DZMQ_STATIC)
ENDIF()

find_package(nlohmann_json QUIET)
IF ( nlohmann_json_FOUND)
    message("USING system nlohmann_json")
ELSE()
    message("USING third_party/nlohmann_json")
    SET(nlohmann_json_FOUND TRUE)
    set(nlohmann_json_INCLUDE_DIRS ../third_party/nlohmann/include)
    include_directories(${nlohmann_json_INCLUDE_DIRS})
ENDIF()

#find_library(zmq_location NAMES libzmq.a)
#message(STATUS ${zmq_location})
#add_library(zmq STATIC IMPORTED)
#set_target_properties(zmq PROPERTIES IMPORTED_LOCATION ${zmq_location})

# show all cmake variables, for debugging
# get_cmake_property(_variableNames VARIABLES)
# list (SORT _variableNames)
# foreach (_variableName ${_variableNames})
#    message(STATUS "${_variableName}=${${_variableName}}")
# endforeach()



set(MESHCAT_LIBRARIES crossguid cpp_base64 zmq )
IF(WIN32)
    set(MESHCAT_LIBRARIES ${MESHCAT_LIBRARIES} ws2_32 Winmm iphlpapi)
ENDIF()
set(HAS_MESHCAT TRUE)

pkg_check_modules(bullet REQUIRED bullet>=2.89)

if (bullet_FOUND)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
    include_directories(${bullet_INCLUDE_DIRS})
    link_directories(${bullet_LIBRARY_DIRS})
    add_library(pybullet_visualizer_api STATIC pybullet_visualizer_api.cpp)
    target_link_libraries(pybullet_visualizer_api BulletRoboticsGUI BulletExampleBrowserLib
            BulletRobotics BulletFileLoader BulletWorldImporter BulletSoftBody BulletDynamics
            BulletCollision BulletInverseDynamicsUtils BulletInverseDynamics LinearMath
            OpenGLWindow gwen BussIK Bullet3Common pthread ${DL})

    if (Ceres_FOUND)
        include_directories(${CERES_INCLUDE_DIRS})
        add_executable(billiard_opt_gui billiard_opt_gui.cpp ${TDS_HDRS})
        target_link_libraries(billiard_opt_gui pybullet_visualizer_api ${DL}
                ${CERES_LIBRARIES})
        target_compile_definitions(billiard_opt_gui PRIVATE STAN_MATH)

        add_executable(cartpole_opt cartpole_opt.cpp ${TDS_HDRS})
        target_link_libraries(cartpole_opt pybullet_visualizer_api ${DL}
                ${CERES_LIBRARIES})

        add_executable(ceres_estimation ceres_estimation.cpp ${TDS_HDRS})
        target_link_libraries(ceres_estimation pybullet_visualizer_api ${DL}
                ${CERES_LIBRARIES})

        add_executable(policy_optimization policy_optimization.cpp ${TDS_HDRS})
        target_link_libraries(policy_optimization pybullet_visualizer_api ${DL}
                ${CERES_LIBRARIES})

        add_executable(contact_discovery contact_discovery.cpp ${TDS_HDRS})
        target_link_libraries(contact_discovery pybullet_visualizer_api ${DL}
                ${CERES_LIBRARIES})
    endif (Ceres_FOUND)

    if (ct_core_FOUND AND ct_optcon_FOUND AND Eigen3_FOUND)
        # Activate CppAD (with CodeGen) and HPIPM for Control Toolbox
        add_definitions(-DCPPAD -DCPPADCG -DHPIPM)
    
        include_directories(${ct_core_INCLUDE_DIRS})
        include_directories(${ct_optcon_INCLUDE_DIRS})
        include_directories(${EIGEN3_INCLUDE_DIRS})

        add_executable(ct_cartpole ct_cartpole.cpp ${TDS_HDRS})
        target_link_libraries(ct_cartpole pybullet_visualizer_api ${DL}
                -lpthread hpipm blasfeo)

        add_executable(ct_contact_trajopt ct_contact_trajopt.cpp ${TDS_HDRS})
        target_link_libraries(ct_contact_trajopt pybullet_visualizer_api ${DL}
                -lpthread hpipm blasfeo)

        add_executable(ct_laikago_tracking ct_laikago_tracking.cpp ${TDS_HDRS})
        target_link_libraries(ct_laikago_tracking pybullet_visualizer_api ${DL}
                -lpthread hpipm blasfeo)
    endif (ct_core_FOUND AND ct_optcon_FOUND AND Eigen3_FOUND)

    add_executable(laikago_example_gui laikago_example_gui.cpp ${TDS_HDRS})
    target_link_libraries(laikago_example_gui pybullet_visualizer_api)

    if (Eigen3_FOUND)
      add_executable(laikago_footstep_ik laikago_footstep_ik.cpp ${TDS_HDRS})
      target_link_libraries(laikago_footstep_ik pybullet_visualizer_api)

      add_executable(pendulum_ik_gui pendulum_ik_gui.cpp)
      target_link_libraries(pendulum_ik_gui pybullet_visualizer_api)
    endif()

    add_executable(pendulum_example_gui pendulum_example_gui.cpp ${TDS_HDRS})
    target_link_libraries(pendulum_example_gui pybullet_visualizer_api)

    add_executable(generic_urdf_collision generic_urdf_collision.cpp ${TDS_HDRS})
    target_link_libraries(generic_urdf_collision pybullet_visualizer_api)

    add_executable(gravity_compensation gravity_compensation.cpp ${TDS_HDRS})
    target_link_libraries(gravity_compensation pybullet_visualizer_api)

if (HAS_MESHCAT)
    add_executable(laikago_meshcat_example laikago_meshcat_example.cpp ${TDS_HDRS})
    target_link_libraries(laikago_meshcat_example  pybullet_visualizer_api ${MESHCAT_LIBRARIES})
    
endif (HAS_MESHCAT)
endif (bullet_FOUND)

if (HAS_MESHCAT AND tinyxml2_FOUND)
    add_executable(tiny_urdf_parser_meshcat_example tiny_urdf_parser_meshcat_example.cpp ${TDS_HDRS}) 
    target_link_libraries(tiny_urdf_parser_meshcat_example  tinyxml2 ${MESHCAT_LIBRARIES}) 


    add_executable(meshcat_zmq_example meshcat_zmq_example.cpp ${TDS_HDRS})
    target_link_libraries(meshcat_zmq_example  ${MESHCAT_LIBRARIES})

endif (HAS_MESHCAT AND tinyxml2_FOUND)

if (tinyxml2_FOUND)
    add_executable(tiny_urdf_parser_example tiny_urdf_parser_example.cpp ${TDS_HDRS})
    target_link_libraries(tiny_urdf_parser_example tinyxml2)
endif (tinyxml2_FOUND)

add_executable(world_example world_example.cpp ${TDS_HDRS})
add_executable(test_xarm test_xarm.cpp ${TDS_HDRS})
