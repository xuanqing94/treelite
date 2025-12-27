cmake_policy(SET CMP0025 NEW)
cmake_policy(SET CMP0091 NEW)
set(CMAKE_FIND_NO_INSTALL_PREFIX TRUE FORCE)
cmake_minimum_required(VERSION 3.16 FATAL_ERROR)

project(treelite LANGUAGES CXX C VERSION 4.7.0)
if(POLICY CMP0135)
  cmake_policy(SET CMP0135 NEW)
endif()


# Options useful for downstream projects
option(Treelite_BUILD_STATIC_LIBS "Build static libs, in addition to dynamic libs" OFF)
option(Treelite_USE_DYNAMIC_MSVC_RUNTIME
    "Whether to dynamically link MSVC runtime. ON: Use dynamic link (-MD), OFF: Use static link (-MT)"
    OFF)

# Developer options
option(TEST_COVERAGE "C++ test coverage" OFF)
option(USE_OPENMP "Use OpenMP" ON)
option(BUILD_DOXYGEN "Build documentation for C/C++ functions using Doxygen." OFF)
option(DETECT_CONDA_ENV "Enable detection of conda environment for dependencies" ON)
option(HIDE_CXX_SYMBOLS "Hide all C++ symbols. Useful when building Pip package" OFF)
option(ENABLE_ALL_WARNINGS "Enable all compiler warnings. Only effective for GCC/Clang" OFF)
option(USE_SANITIZER "Use sanitizer flags" OFF)
SET(ENABLED_SANITIZERS "address" "leak" "undefined" CACHE STRING
    "Semicolon separated list of sanitizer names. E.g 'address;leak'.")

if(USE_SANITIZER)
  include(cmake/Sanitizer.cmake)
  enable_sanitizers("${ENABLED_SANITIZERS}")
endif()

if(ENABLE_ALL_WARNINGS)
  if((NOT CMAKE_CXX_COMPILER_ID MATCHES "Clang") AND (NOT CMAKE_CXX_COMPILER_ID STREQUAL "GNU"))
    message(SEND_ERROR "ENABLE_ALL_WARNINGS is only available for Clang and GCC.")
  endif()
endif()

if(BUILD_DOXYGEN)
  include(cmake/Doxygen.cmake)
  run_doxygen()
endif()

include(cmake/ExternalLibs.cmake)
include(cmake/Utils.cmake)
include(cmake/Version.cmake)

set(TREELITE_SOURCES
    # Headers
    ${PROJECT_SOURCE_DIR}/include/treelite/c_api.h
    ${PROJECT_SOURCE_DIR}/include/treelite/c_api_error.h
    ${PROJECT_SOURCE_DIR}/include/treelite/contiguous_array.h
    ${PROJECT_SOURCE_DIR}/include/treelite/error.h
    ${PROJECT_SOURCE_DIR}/include/treelite/gtil.h
    ${PROJECT_SOURCE_DIR}/include/treelite/logging.h
    ${PROJECT_SOURCE_DIR}/include/treelite/model_builder.h
    ${PROJECT_SOURCE_DIR}/include/treelite/model_loader.h
    ${PROJECT_SOURCE_DIR}/include/treelite/pybuffer_frame.h
    ${PROJECT_SOURCE_DIR}/include/treelite/thread_local.h
    ${PROJECT_SOURCE_DIR}/include/treelite/tree.h
    ${PROJECT_SOURCE_DIR}/include/treelite/detail/contiguous_array.h
    ${PROJECT_SOURCE_DIR}/include/treelite/detail/file_utils.h
    ${PROJECT_SOURCE_DIR}/include/treelite/detail/omp_exception.h
    ${PROJECT_SOURCE_DIR}/include/treelite/detail/serializer.h
    ${PROJECT_SOURCE_DIR}/include/treelite/detail/serializer_mixins.h
    ${PROJECT_SOURCE_DIR}/include/treelite/detail/threading_utils.h
    ${PROJECT_SOURCE_DIR}/include/treelite/detail/tree.h
    ${PROJECT_SOURCE_DIR}/include/treelite/enum/operator.h
    ${PROJECT_SOURCE_DIR}/include/treelite/enum/task_type.h
    ${PROJECT_SOURCE_DIR}/include/treelite/enum/tree_node_type.h
    ${PROJECT_SOURCE_DIR}/include/treelite/enum/typeinfo.h
    
    # Sources (Paths relative to root usually, unless src/ handles it)
    # Assuming these files are inside 'src/' folder, adjust paths if needed.
    src/field_accessor.cc
    src/json_serializer.cc
    src/logging.cc
    src/model_concat.cc
    src/model_query.cc
    src/serializer.cc
    src/c_api/c_api_error.cc
    src/c_api/c_api_utils.h
    src/c_api/field_accessor.cc
    src/c_api/gtil.cc
    src/c_api/logging.cc
    src/c_api/model.cc
    src/c_api/model_builder.cc
    src/c_api/model_loader.cc
    src/c_api/serializer.cc
    src/c_api/sklearn.cc
    src/enum/operator.cc
    src/enum/task_type.cc
    src/enum/tree_node_type.cc
    src/enum/typeinfo.cc
    src/gtil/config.cc
    src/gtil/output_shape.cc
    src/gtil/postprocessor.cc
    src/gtil/postprocessor.h
    src/gtil/predict.cc
    src/model_builder/metadata.cc
    src/model_builder/model_builder.cc
    src/model_builder/detail/json_parsing.h
    src/model_loader/lightgbm.cc
    src/model_loader/sklearn.cc
    src/model_loader/xgboost_json.cc
    src/model_loader/xgboost_legacy.cc
    src/model_loader/xgboost_ubjson.cc
    src/model_loader/detail/lightgbm.h
    src/model_loader/detail/string_utils.h
    src/model_loader/detail/xgboost.cc
    src/model_loader/detail/xgboost.h
    src/model_loader/detail/xgboost_json/sax_adapters.h
    src/model_loader/detail/xgboost_json/sax_adapters.cc
    src/model_loader/detail/xgboost_json/delegated_handler.h
    src/model_loader/detail/xgboost_json/delegated_handler.cc
)

function(setup_treelite_target target_name)
    # -- 1. Link Internal Dependencies (PRIVATE) --
    # Crucial: This hides json/mdspan from the export set.
    target_link_libraries(${target_name} 
        PRIVATE 
        RapidJSON::rapidjson 
        nlohmann_json::nlohmann_json 
        std::mdspan
    )

    # -- 2. Include Directories (PUBLIC) --
    # This ensures consumers can find treelite/c_api.h
    target_include_directories(${target_name} PUBLIC
        $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
        $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    )

    # -- 3. Compiler Options --
    target_compile_features(${target_name} PUBLIC cxx_std_17)
    set_target_properties(${target_name} PROPERTIES POSITION_INDEPENDENT_CODE ON)
    
    if(ENABLE_ALL_WARNINGS)
        target_compile_options(${target_name} PRIVATE -Wall -Wextra)
    endif()

    target_compile_options(${target_name} PRIVATE -funroll-loops)
    if(HIDE_CXX_SYMBOLS)
        set_target_properties(${target_name} PROPERTIES CXX_VISIBILITY_PRESET hidden)
    endif()

    if(TEST_COVERAGE AND NOT MSVC)
        target_compile_options(${target_name} PUBLIC -g3 --coverage)
        target_link_options(${target_name} PUBLIC --coverage)
    endif()

    # -- 4. OpenMP --
    if(USE_OPENMP)
        find_package(OpenMP REQUIRED)
        if(OpenMP_FOUND)
            target_link_libraries(${target_name} PRIVATE OpenMP::OpenMP_CXX)
            target_compile_definitions(${target_name} PUBLIC -DTREELITE_OPENMP_SUPPORT)
        endif()
    endif()
endfunction()

add_library(treelite SHARED ${TREELITE_SOURCES})
setup_treelite_target(treelite)
set_target_properties(treelite PROPERTIES VERSION ${PROJECT_VERSION})

if(Treelite_BUILD_STATIC_LIBS)
    add_library(treelite_static STATIC ${TREELITE_SOURCES})
    setup_treelite_target(treelite_static)
    # Ensure standard naming (libtreelite.a)
    set_target_properties(treelite_static PROPERTIES OUTPUT_NAME "treelite")
endif()
write_version()
