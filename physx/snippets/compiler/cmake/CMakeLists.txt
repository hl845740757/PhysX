## Redistribution and use in source and binary forms, with or without
## modification, are permitted provided that the following conditions
## are met:
##  * Redistributions of source code must retain the above copyright
##    notice, this list of conditions and the following disclaimer.
##  * Redistributions in binary form must reproduce the above copyright
##    notice, this list of conditions and the following disclaimer in the
##    documentation and/or other materials provided with the distribution.
##  * Neither the name of NVIDIA CORPORATION nor the names of its
##    contributors may be used to endorse or promote products derived
##    from this software without specific prior written permission.
##
## THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS ''AS IS'' AND ANY
## EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
## IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
## PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
## CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
## EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
## PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
## PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY
## OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
## (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
## OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
##
## Copyright (c) 2008-2025 NVIDIA Corporation. All rights reserved.

cmake_minimum_required(VERSION 3.16)
project(Snippets C CXX)

# This is required to be defined by external callers!
IF(NOT DEFINED PHYSX_ROOT_DIR)
	MESSAGE(FATAL_ERROR "PHYSX_ROOT_DIR variable wasn't set.")
ENDIF()

IF(NOT EXISTS ${PHYSX_ROOT_DIR})
	MESSAGE(FATAL_ERROR "PHYSX_ROOT_DIR variable was invalid.")
ENDIF()

INCLUDE(NvidiaBuildOptions)


IF(CMAKE_CONFIGURATION_TYPES)
	SET(CMAKE_CONFIGURATION_TYPES debug checked profile release)
	SET(CMAKE_CONFIGURATION_TYPES "${CMAKE_CONFIGURATION_TYPES}" CACHE STRING
		"Reset config to what we need"
		FORCE)

	# Need to define these at least once.
	SET(CMAKE_SHARED_LINKER_FLAGS_CHECKED "/DEBUG")
	SET(CMAKE_SHARED_LINKER_FLAGS_PROFILE "/DEBUG")

	SET(CMAKE_EXE_LINKER_FLAGS_PROFILE "/DEBUG")
	SET(CMAKE_EXE_LINKER_FLAGS_CHECKED "/DEBUG")
ENDIF()


SET(PROJECT_CMAKE_FILES_DIR compiler/cmake)
SET(PLATFORM_CMAKELISTS ${PHYSX_ROOT_DIR}/snippets/${PROJECT_CMAKE_FILES_DIR}/${TARGET_BUILD_PLATFORM}/CMakeLists.txt)

IF(NOT EXISTS ${PLATFORM_CMAKELISTS})
	MESSAGE(FATAL_ERROR "Unable to find platform CMakeLists.txt for ${TARGET_BUILD_PLATFORM} at ${PLATFORM_CMAKELISTS}")
ENDIF()

# Include the platform specific CMakeLists
INCLUDE(${PHYSX_ROOT_DIR}/snippets/${PROJECT_CMAKE_FILES_DIR}/${TARGET_BUILD_PLATFORM}/CMakeLists.txt)

SET(SOURCE_DISTRO_FILE_LIST "")

# Include all of the projects
SET(SNIPPETS_LIST ArticulationRC BVHStructure CCD ContactModification ContactReport ContactReportCCD ConvexMeshCreate
	CustomJoint CustomProfiler DeformableMesh FrustumQuery GearJoint GeometryQuery Gyroscopic HelloWorld ImmediateArticulation ImmediateMode Joint JointDrive MassProperties
	MBP MimicJoint MultiPruners MultiThreading OmniPvd PathTracing PointDistanceQuery ProfilerConverter PrunerSerialization QuerySystemAllQueries QuerySystemCustomCompound RackJoint Serialization SplitFetchResults
	SplitSim StandaloneBVH StandaloneBroadphase StandaloneQuerySystem Stepper ToleranceScale TriangleMeshCreate Triggers CustomGeometry CustomConvex CustomGeometryCollision CustomGeometryQueries FixedTendon SpatialTendon)
LIST(APPEND SNIPPETS_LIST ${PLATFORM_SNIPPETS_LIST})

# Add further snippets that use GPU features directly.
# These are separate because we don't ship CUDA in the public release yet.
# To be changed if we decide otherwise.
SET(GPU_SNIPPET_LIST
	HelloGRB DeformableSurface DeformableVolume DeformableVolumeAttachment DeformableVolumeKinematic DeformableSurfaceSkinning DeformableVolumeSkinning SDF PBF PBFMultiMat PBDCloth PBDInflatable Isosurface
	RBDirectGPUAPI DirectGPUAPIArticulation)

#TODO, create a propper define for whether GPU features are enabled or not!
IF (PX_GENERATE_GPU_PROJECTS AND 
    (NOT (CMAKE_CROSSCOMPILING OR CMAKE_GENERATOR_PLATFORM STREQUAL "NX64")))
    LIST(APPEND SNIPPETS_LIST ${GPU_SNIPPET_LIST})
ENDIF()
	
SET(SNIPPETS_VEHICLE_LIST VehicleDirectDrive VehicleFourWheelDrive VehicleTankDrive VehicleCustomSuspension VehicleMultithreading VehicleTruck VehicleCustomTire)
LIST(APPEND SNIPPETS_VEHICLE_LIST ${PLATFORM_SNIPPETS_VEHICLE_LIST})
	
IF(SNIPPET_RENDER_ENABLED)
	INCLUDE(SnippetRender.cmake)
	SET_PROPERTY(TARGET SnippetRender PROPERTY FOLDER "Snippets/Libraries only")
ENDIF()

INCLUDE(SnippetUtils.cmake)
SET_PROPERTY(TARGET SnippetUtils PROPERTY FOLDER "Snippets/Libraries only")

# Compile snippets
FOREACH(SNIPPET_NAME ${SNIPPETS_LIST})
	INCLUDE(SnippetTemplate.cmake)
	SET_PROPERTY(TARGET Snippet${SNIPPET_NAME} PROPERTY FOLDER "Snippets")
ENDFOREACH()

# Compile vehicle snippets
FOREACH(SNIPPET_NAME ${SNIPPETS_VEHICLE_LIST})
	INCLUDE(SnippetVehicleTemplate.cmake)
	SET_PROPERTY(TARGET Snippet${SNIPPET_NAME} PROPERTY FOLDER "Snippets")
ENDFOREACH()

# For internal builds, make sure the GPU lib is built (but not linked!) before GPU snippets
IF (PX_GENERATE_GPU_PROJECTS OR PX_GENERATE_GPU_PROJECTS_ONLY)
	FOREACH(SNIPPET_NAME ${GPU_SNIPPET_LIST})
		ADD_DEPENDENCIES(Snippet${SNIPPET_NAME} PhysXGpu)
	ENDFOREACH()
ENDIF()

IF(PX_GENERATE_SOURCE_DISTRO)
	FOREACH(FILE_NAME ${SOURCE_DISTRO_FILE_LIST})
		FILE(APPEND "${CMAKE_CURRENT_BINARY_DIR}/source_distro_list.txt" "${FILE_NAME}\n")
	ENDFOREACH()
ENDIF()
