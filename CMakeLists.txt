cmake_minimum_required(VERSION 3.4.0)

set(LULESH_SOURCES
        lulesh.cc
        lulesh-comm.cc
        lulesh-init.cc
        lulesh-util.cc
        lulesh-viz.cc)

project(LULESH CXX)

option(WITH_MPI "Build LULESH with MPI" FALSE)
option(WITH_OPENMP "Build LULESH with OpenMP" TRUE)
option(WITH_SILO "Build LULESH with silo support" FALSE)

if (WITH_MPI)
    message("-- Using MPI")
    add_definitions("-DUSE_MPI=1")
    find_package(MPI QUIET)
    if (NOT MPI_FOUND)
        set(MPI_FAASM_DIR /usr/local/faasm/openmpi)
        set(MPI_DIR ${MPI_FAASM_DIR})
        set(MPI_C_INCLUDE_PATH ${MPI_FAASM_DIR}/include)
        set(MPI_CXX_INCLUDE_PATH ${MPI_FAASM_DIR}/include)
        set(MPI_C_LIBRARIES ${MPI_FAASM_DIR}/lib/libmpi.so)
        set(MPI_CXX_LIBRARIES ${MPI_FAASM_DIR}/lib/libmpi.so)
    endif ()
    include_directories(${MPI_C_INCLUDE_PATH} ${MPI_CXX_INCLUDE_PATH})
else ()
    add_definitions("-DUSE_MPI=0")
endif ()

if (FAASM_BUILD_TYPE STREQUAL "wasm")
    # In sync with Faasm func
    SET(CMAKE_EXE_LINKER_FLAGS "${FAASM_EXE_LINKER_FLAGS} \
        -Xlinker --export=_faasm_zygote \
        -Xlinker --export=__heap_base \
        -Xlinker --export=__data_end \
        -Xlinker --export=__wasm_call_ctors \
        -Xlinker --max-memory=4294901760 \
        -Wl,-z,stack-size=4194304 -Wl, \
    ")

    if (WITH_MPI AND WITH_OPENMP)
        set(lulesh_exec lulesh_ompi)
    elseif (WITH_MPI)
        set(lulesh_exec lulesh_mpi)
    elseif (WITH_OPENMP)
        set(lulesh_exec lulesh_omp)
    else ()
        set(lulesh_exec lulesh_bare)
    endif ()

    SET(CMAKE_EXECUTABLE_SUFFIX ".wasm")
    add_executable(${lulesh_exec} ${LULESH_SOURCES})
    target_link_libraries(${lulesh_exec} faasm dlmalloc)

    if (WITH_OPENMP)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fopenmp")
        target_link_libraries(${lulesh_exec} faasmp)
    endif ()
    if (WITH_MPI)
        target_link_libraries(${lulesh_exec} faasmpi)
    endif ()

else ()
    set(LULESH_EXEC lulesh2.0)

    if (WITH_MPI)
        list(APPEND LULESH_EXTERNAL_LIBS ${MPI_C_LIBRARIES} ${MPI_CXX_LIBRARIES})
    endif ()

    if (WITH_OPENMP)
        find_package(OpenMP REQUIRED)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
    endif ()

    if (WITH_SILO)
        find_path(SILO_INCLUDE_DIR silo.h
                HINTS ${SILO_DIR}/include)
        find_library(SILO_LIBRARY
                NAMES siloh5
                HINTS ${SILO_DIR}/lib)

        include(FindPackageHandleStandardArgs)
        find_package_handle_standard_args(SILO DEFAULT_MSG
                SILO_LIBRARY
                SILO_INCLUDE_DIR)

        if (SILO_FOUND)
            add_definitions("-DVIZ_MESH")
            include_directories(${SILO_INCLUDE_DIR})
            #   Note: silo needs to be built as a dynamic lib, otherwise
            # there are additional dependencies (hdf5) which we don't know.
            # This would be fixed by silo providing a CMake package.
            list(APPEND LULESH_EXTERNAL_LIBS ${SILO_LIBRARY})
        endif ()
    endif ()

#    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address -fno-omit-frame-pointer")
    add_executable(${LULESH_EXEC} ${LULESH_SOURCES})
    target_link_libraries(${LULESH_EXEC} ${LULESH_EXTERNAL_LIBS})
endif ()
