cmake_minimum_required(VERSION 3.12...4.0)

project(gau2grid
        VERSION 2.0.8
        LANGUAGES C)
set(gau2grid_AUTHORS      "Daniel G. A. Smith")
set(gau2grid_DESCRIPTION  "Fast computation of a gaussian and its derivative on a grid")
set(gau2grid_URL          "https://github.com/dgasmith/gau2grid")
set(gau2grid_LICENSE      "BSD 3-clause")

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)


#############################  Options: Build How?  #############################
include(GNUInstallDirs)
include(psi4OptionsTools)
option_with_default(MAX_AM "The maximum gaussian angular momentum to compile" 8)
option_with_default(CMAKE_BUILD_TYPE "Build type (Release or Debug)" Release)
if(CMAKE_CXX_COMPILER_ID MATCHES Intel)
option_with_flags(ENABLE_XHOST "Enables processor-specific optimization (with MSVC, it enables AVX2 instructions)" ON
                  "-xHost" "-march=native" "/arch:AVX2")
else()
option_with_flags(ENABLE_XHOST "Enables processor-specific optimization (with MSVC, it enables AVX2 instructions)" ON
                  "-march=native" "-xHost" "/arch:AVX2")
endif()
option_with_default(BUILD_FPIC "Libraries will be compiled with position independent code" ON)
option_with_print(BUILD_SHARED_LIBS "Build final library as shared, not static" ON)
option_with_default(ENABLE_GENERIC "Enables mostly static linking of system libraries for shared library" OFF)
option_with_default(DISABLE_PRAGMA "Disable certain pragma optimizations, appends _GG_NO_PRAGMA to compile flags" OFF )

# Warnings
if((${BUILD_SHARED_LIBS}) AND NOT ${BUILD_FPIC})
    message(FATAL_ERROR "BUILD_SHARED_LIBS ON and BUILD_FPIC OFF are incompatible, as shared library requires position independent code")
endif()

# Install
option_with_default(PYMOD_INSTALL_LIBDIR "Location within CMAKE_INSTALL_LIBDIR to which python modules are installed
                                          Must start with: / . Used to imitate python install: /python3.6/site-packages ." /)
option_with_print(INSTALL_PYMOD "Additionally installs as independent python module in PYMOD_INSTALL_LIBDIR" OFF)
option_with_default(NATIVE_PYTHON_INSTALL "For INSTALL_PYMOD=ON, install in Python manner to PYTHON_EXECUTABLE's site-packages rather than Linux manner to prefix. Overrides CMAKE_INSTALL_PREFIX, CMAKE_INSTALL_LIBDIR, PYMOD_INSTALL_LIBDIR. Only Py module installed." OFF)
option_with_print(NATIVE_PYTHON_INSTALL_WITH_LIB "Same as NATIVE_PYTHON_INSTALL except installs library, too, _without_ overriding CMAKE_INSTALL_* options." OFF)

########################  Process & Validate Options  ##########################
include(autocmake_safeguards)
include(custom_color_messages)
include(custom_static_library)

#  <<  Python  >>
find_package(Python 3.6 REQUIRED COMPONENTS Interpreter)
message(STATUS "${Cyan}Found Python ${Python_VERSION_MAJOR}.${Python_VERSION_MINOR}${ColourReset}: ${Python_EXECUTABLE} (found version ${Python_VERSION})")


################################  Main Project  ################################
add_custom_command(
    OUTPUT  gau2grid/gau2grid.h gau2grid_orbital.c gau2grid_phi.c gau2grid_deriv1.c gau2grid_deriv2.c gau2grid_deriv3.c gau2grid_transform.c gau2grid_helper.c
    COMMAND Python::Interpreter -c "import sys; \
                                     sys.path.append('${PROJECT_SOURCE_DIR}'); \
                                     import gau2grid as gg; \
                                     gg.c_gen.generate_c_gau2grid(${MAX_AM}, path='${CMAKE_CURRENT_BINARY_DIR}')"
    DEPENDS gau2grid/c_generator.py
            gau2grid/c_generator.py
            gau2grid/codegen.py
            gau2grid/c_pragma.py
            gau2grid/c_util_generator.py
            gau2grid/c_wrapper.py
            gau2grid/docs_generator.py
            gau2grid/order.py
            gau2grid/python_reference.py
            gau2grid/RSH.py
            gau2grid/utility.py
    VERBATIM)

set(sources_list ${CMAKE_CURRENT_BINARY_DIR}/gau2grid_phi.c
                 ${CMAKE_CURRENT_BINARY_DIR}/gau2grid_orbital.c
                 ${CMAKE_CURRENT_BINARY_DIR}/gau2grid_deriv1.c
                 ${CMAKE_CURRENT_BINARY_DIR}/gau2grid_deriv2.c
                 ${CMAKE_CURRENT_BINARY_DIR}/gau2grid_deriv3.c
                 ${CMAKE_CURRENT_BINARY_DIR}/gau2grid_transform.c
                 ${CMAKE_CURRENT_BINARY_DIR}/gau2grid_helper.c)

add_library(gg ${sources_list})
target_include_directories(gg PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_include_directories(gg PRIVATE ${PROJECT_SOURCE_DIR})
if ("${CMAKE_C_COMPILER_ID}" STREQUAL "PGI")
    set_target_properties(gg PROPERTIES COMPILE_FLAGS "-c11")
else()
    set_target_properties(gg PROPERTIES COMPILE_FLAGS "-std=c11")
endif()
set_target_properties(gg PROPERTIES POSITION_INDEPENDENT_CODE ${BUILD_FPIC}
                                    SOVERSION 2)  # bump whenever interface has changes or removals

if( DISABLE_PRAGMA )
  target_compile_definitions( gg PRIVATE $<BUILD_INTERFACE:__GG_NO_PRAGMA> )
endif()

find_package(StandardMathLibraryC)
target_link_libraries(gg PRIVATE ${STANDARD_MATH_LIBRARY})

if(${BUILD_SHARED_LIBS})
    target_link_libraries(gg PRIVATE ${LIBC_INTERJECT})
endif()