cmake_minimum_required(VERSION 3.15)
project(r8brain)

option(R8B_EXTFFT "Enable FFT extension" OFF)

option(R8B_FASTTIMING "Enable fast interpolation sample timing technique" OFF)

option(R8B_IPP "Enable Intel Performance Primitives support" OFF)



# Define source files
set(SOURCE_FILES
        r8bbase.cpp
        pffft.cpp
        pffft_double/pffft_double.c
)

# Define header files
set(PRIVATE_HEADER_FILES
        CDSPHBDownsampler.inc
        CDSPHBUpsampler.inc
        fft4g.h
        pffft.h
        pffft_double/pffft_double.h
        pffft_double/pffft_priv_impl.h
        pffft_double/simd/pf_avx_double.h
        pffft_double/simd/pf_double.h
        pffft_double/simd/pf_neon_double.h
        pffft_double/simd/pf_neon_double_from_avx.h
        pffft_double/simd/pf_scalar_double.h
        pffft_double/simd/pf_sse2_double.h
        r8bbase.h
        r8bconf.h
        r8butil.h
)

set(PUBLIC_HEADER_FILES
        CDSPBlockConvolver.h
        CDSPFIRFilter.h
        CDSPFracInterpolator.h
        CDSPHBDownsampler.h
        CDSPHBUpsampler.h
        CDSPProcessor.h
        CDSPRealFFT.h
        CDSPResampler.h
        CDSPSincFilterGen.h
)

# Add library target
add_library(${PROJECT_NAME} ${SOURCE_FILES})

# Set target include directories
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:include>
        SYSTEM PRIVATE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/DLL>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/bench>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/other>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/pffft_double>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/pffft_double/simd>
)

# Make public headers available to other targets
set_target_properties(${PROJECT_NAME} PROPERTIES PUBLIC_HEADER "${PUBLIC_HEADER_FILES}")


# Set compile flags

if(R8B_EXTFFT)
    message("r8brain using FFT extension")
    target_compile_definitions(${PROJECT_NAME} PUBLIC R8B_EXTFFT=1)
endif()

if(R8B_FASTTIMING)
    message("r8brain using fast timing")
    target_compile_definitions(${PROJECT_NAME} PUBLIC R8B_FASTTIMING=1)
endif()

if (R8B_IPP)
    # setup IPP if available, otherwise use PFFFT
    find_package(IPP)
    if(IPP_FOUND)
        target_link_libraries(${PROJECT_NAME} PRIVATE IPP::ipps IPP::ippcore IPP::ippi IPP::ippcv)
        target_compile_definitions(${PROJECT_NAME} PUBLIC R8B_IPP=1)
        message("r8brain using IPP")
    else() 
        message(WARNING "R8B_IPP enabled but IPP not found. Using PFFFT instead.")
        target_compile_definitions(${PROJECT_NAME} PUBLIC R8B_PFFFT_DOUBLE=1)
    endif()
else() 
    target_compile_definitions(${PROJECT_NAME} PUBLIC R8B_PFFFT_DOUBLE=1)
endif()

# Link dependencies
if (UNIX)
    find_package(Threads REQUIRED)
    target_link_libraries(${PROJECT_NAME} PRIVATE Threads::Threads)
elseif (WIN32)
    target_link_libraries(${PROJECT_NAME} PRIVATE Kernel32)
endif()


# Specify installation for the library (including public headers)
install(TARGETS ${PROJECT_NAME} 
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
        PUBLIC_HEADER DESTINATION include/r8brain
)
