# Set the minimum required version of cmake for this project.
cmake_minimum_required(VERSION 3.4)

# Fix cmake behavior for CMAKE_CXX_STANDARD with clang on Mac OS X.
# See cmake documentation on CMP0025 policy for details.
# Must be done before doing project() command.
if(POLICY CMP0025)
    cmake_policy(SET CMP0025 NEW)
endif()

# Set policy so we can use some of the recent Jed Brown Find*.cmake files.
if(POLICY CMP0057)
    cmake_policy(SET CMP0057 NEW)
endif()

# Set a name for this project.
PROJECT ( xolotl C CXX )

# Build the tests by default
option(BUILD_TESTS "Build test programs" ON)

# Include the modules to find PETSC, ITAPS, etc. from Jed Brown
list (APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake-modules")

# xolotl requires some C++11 features.
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Tell CMake to look for static libraries
# SET(BUILD_SHARED_LIBS OFF)

# Find PETSc
# We need to save and restore CMAKE_REQUIRED_{INCLUDES,LIBRARIES}
# because the FindPETSc.cmake module sets during its operation
# but does not reset them, and the leftover state then messes with 
# our subsequent configuration checks.
set(SAVED_CMAKE_REQUIRED_INCLUDES ${CMAKE_REQUIRED_INCLUDES})
set(SAVED_CMAKE_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES})
FIND_PACKAGE(PETSc REQUIRED)
if (PETSC_FOUND)
    include_directories(${PETSC_INCLUDES})
    set(CMAKE_REQUIRED_INCLUDES ${SAVED_CMAKE_REQUIRED_INCLUDES})
    set(CMAKE_REQUIRED_LIBRARIES ${SAVED_CMAKE_REQUIRED_LIBRARIES})
endif()

# Make sure that we only pull the static HDF5
#set(HDF5_USE_STATIC_LIBRARIES yes)
# Find HDF5 - Optional for now until we actually require it
FIND_PACKAGE(HDF5 REQUIRED)
# Check for parallel I/O support in the HDF5 installation.
IF (NOT HDF5_IS_PARALLEL)
   message(STATUS "WARNING! Detected HDF5 installation does not support parallel I/O!")
ENDIF()
# Include the headers
INCLUDE_DIRECTORIES(${HDF5_INCLUDE_DIR})

# Enable testing.
enable_testing()

# Find the Boost libraries we (potentially) use.
# Note that we only need to list Boost component libraries that have a
# library implementation (i.e., not header only) as required components.
FIND_PACKAGE(Boost OPTIONAL_COMPONENTS chrono timer filesystem unit_test_framework)
IF (Boost_FOUND)
    message(STATUS "Boost version ${Boost_VERSION} found.")
    INCLUDE_DIRECTORIES(${Boost_INCLUDE_DIR})
ENDIF()
IF (Boost_UNIT_TEST_FRAMEWORK_FOUND)
    IF (BUILD_TESTS)
        # Enable testing.
        enable_testing()
    	message(STATUS "Tests will be built since Boost unit test framework found.")
    	add_subdirectory(tests)
	ENDIF()
ENDIF()

# Add the preprocessor first, it is independent of the other parts of the code
# But only if Java is present
FIND_PACKAGE(Java)
IF (Java_FOUND)
    message(STATUS "The preprocessor will be built since Java was found.")
    add_subdirectory(gov.ornl.xolotl.preprocessor)
ENDIF()

# Build the performance library first, so it can be used by other libraries.
add_subdirectory(xolotlPerf)
# Add the visualization directory
add_subdirectory(xolotlViz)
# Add the I/O, data structures and reactor directories
add_subdirectory(xolotlCore)
# Add the factory directory
add_subdirectory(xolotlFactory)
# Keep the solver for the end (it uses everything else)
add_subdirectory(xolotlSolver)

# Report package information
message(STATUS "----- Configuration Information -----")
message(STATUS "PETSc includes = ${PETSC_INCLUDES}")
message(STATUS "PETSc libraries = ${PETSC_LIBRARIES}")
message(STATUS "----- End Configuration Information -----")

# Create the documentation directory and set up the doc target.
set(DOC_DIR "documentation")
if(NOT EXISTS ${DOC_DIR})
    execute_process(COMMAND ${CMAKE_COMMAND} -E make_directory "${DOC_DIR}")
endif(NOT EXISTS ${DOC_DIR})
set(DOXYFILE_OUTPUT_DIR "${DOC_DIR}")
set(DOXYFILE_LATEX "ON")
set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}")
include(UseDoxygen OPTIONAL)

# Include directories for the main program
include_directories(${CMAKE_SOURCE_DIR}
                    ${CMAKE_SOURCE_DIR}/xolotlCore
                    ${CMAKE_SOURCE_DIR}/xolotlCore/io
                    ${CMAKE_SOURCE_DIR}/xolotlCore/commandline
                    ${CMAKE_SOURCE_DIR}/xolotlCore/commandline/optionhandlers
                    ${CMAKE_SOURCE_DIR}/xolotlCore/diffusion
                    ${CMAKE_SOURCE_DIR}/xolotlCore/advection
                    ${CMAKE_SOURCE_DIR}/xolotlCore/reactants
                    ${CMAKE_SOURCE_DIR}/xolotlCore/reactants/psiclusters
                    ${CMAKE_SOURCE_DIR}/xolotlCore/reactants/neclusters
                    ${CMAKE_SOURCE_DIR}/xolotlCore/temperature
                    ${CMAKE_SOURCE_DIR}/xolotlCore/modifiedreaction/trapmutation
                    ${CMAKE_SOURCE_DIR}/xolotlCore/modifiedreaction/resolution
                    ${CMAKE_SOURCE_DIR}/xolotlCore/flux
                    ${CMAKE_SOURCE_DIR}/xolotlFactory/material
                    ${CMAKE_SOURCE_DIR}/xolotlFactory/vizHandler
                    ${CMAKE_SOURCE_DIR}/xolotlFactory/temperatureHandler
                    ${CMAKE_SOURCE_DIR}/xolotlFactory/solverHandler
                    ${CMAKE_SOURCE_DIR}/xolotlFactory/reactionHandler
                    ${CMAKE_SOURCE_DIR}/xolotlSolver
                    ${CMAKE_SOURCE_DIR}/xolotlPerf
                    ${CMAKE_SOURCE_DIR}/xolotlViz
                    ${CMAKE_BINARY_DIR})

configure_file ("${CMAKE_SOURCE_DIR}/XolotlConfig.h.in" "${CMAKE_BINARY_DIR}/XolotlConfig.h")

# Decide whether to build as a library
option(BUILD_SHARED_LIBS "Build as a shared library")

if(BUILD_SHARED_LIBS)
    message(STATUS "----- Building shared libraries -----")

    # Set the library name
    SET(LIBRARY_NAME "xolotlInter")

    # Collect the header 
    #file(GLOB HEADERS interface.h)
    file(GLOB HEADERS XolotlDLinterface.h)
    # Grab all of the source files
    #file(GLOB SRC interface.cpp)
    file(GLOB SRC DLinterface.cpp)
    #dkim_test
    set(XOLOTL_LIBS xolotlReactants xolotlSolver xolotlIO xolotlPerf xolotlViz 
    xolotlFactory xolotlCL ${PETSC_LIBRARIES} ${HDF5_LIBRARIES})
    # Add a library 
    add_library(${LIBRARY_NAME} SHARED ${SRC})
    #dkim_test
    #target_link_libraries(${LIBRARY_NAME} xolotlFactory)
    target_link_libraries(${LIBRARY_NAME} ${XOLOTL_LIBS})
    
    #Install the xolotl header files
    install(FILES ${HEADERS} DESTINATION ${PROJECT_BINARY_DIR}/include)
    install(TARGETS ${LIBRARY_NAME} DESTINATION ${PROJECT_BINARY_DIR}/lib)
    
else(BUILD_SHARED_LIBS)
    # Set the library name
    SET(LIBRARY_NAME "xolotlInter")

    # Collect the header 
    #file(GLOB HEADERS interface.h)
    file(GLOB HEADERS XolotlDLinterface.h)
    # Grab all of the source files
    #file(GLOB SRC interface.cpp)
    file(GLOB SRC DLinterface.cpp)
    # Add a library 
    add_library(${LIBRARY_NAME} STATIC ${SRC})
    target_link_libraries(${LIBRARY_NAME} xolotlFactory)

    # Setup the library list
    set(XOLOTL_LIBS xolotlReactants xolotlSolver xolotlIO xolotlPerf xolotlViz 
    xolotlFactory xolotlCL xolotlInter ${PETSC_LIBRARIES} ${HDF5_LIBRARIES})
   
    # Add an executable for the main program
    ADD_EXECUTABLE (xolotl main.cpp)

    # Link the libraries
    target_link_libraries(xolotl ${XOLOTL_LIBS})

endif(BUILD_SHARED_LIBS)
