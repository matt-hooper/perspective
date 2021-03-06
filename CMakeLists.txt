cmake_minimum_required (VERSION 2.8.11)
project (psp)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/modules/" ${CMAKE_MODULE_PATH} )


#######################
# BUILD CONFIGURATION #
#######################
find_package(Color)

if (NOT DEFINED PSP_WASM_BUILD)
	set(PSP_WASM_BUILD ON)
	set(PSP_CPP_BUILD OFF)
	set(PSP_CPP_BUILD_TESTS OFF)
	set(PSP_PYTHON_BUILD OFF)
endif()

if (PSP_WASM_BUILD AND PSP_CPP_BUILD)
	message(FATAL_ERROR "${Red}CPP and Emscripten builds must be done separately${ColorReset}")
endif()

if (NOT DEFINED PSP_CPP_BUILD)
	set(PSP_CPP_BUILD ON)
endif()

if (NOT DEFINED PSP_PYTHON_BUILD)
	set(PSP_PYTHON_BUILD OFF)
elseif(PSP_PYTHON_BUILD)
	set(PSP_CPP_BUILD ON)
endif()

if (NOT DEFINED PSP_CPP_BUILD_TESTS)
	set(PSP_CPP_BUILD_TESTS ON)
endif()

if (NOT DEFINED PSP_CPP_BUILD_STRICT)
	set(PSP_CPP_BUILD_STRICT OFF)
endif()

if(PSP_WASM_BUILD)
	message(WARNING "${Cyan}Building WASM binding${ColorReset}")
else()
	message(WARNING "${Cyan}Skipping WASM binding${ColorReset}")
endif()

if(PSP_CPP_BUILD)
	message(WARNING "${Cyan}Building C++ binding${ColorReset}")
else()
	message(WARNING "${Cyan}Skipping C++ binding${ColorReset}")
endif()

if (PSP_PYTHON_BUILD)
	message(WARNING "${Cyan}Building Python binding${ColorReset}")
else()
	message(WARNING "${Cyan}Skipping Python binding${ColorReset}")
endif()

if (PSP_CPP_BUILD_TESTS)
	message(WARNING "${Cyan}Building CPP tests${ColorReset}")
else()
	message(WARNING "${Cyan}Skipping CPP tests${ColorReset}")
endif()

if (NOT PSP_CPP_BUILD_STRICT)
	message(WARNING "${Cyan}Building CPP without strict warnings${ColorReset}")
endif()

#######################

include_directories("${CMAKE_SOURCE_DIR}/src/include")


if (PSP_WASM_BUILD)
	####################
	# EMSCRIPTEN BUILD #
	####################
	add_definitions(-DPSP_ENABLE_WASM)
	set(CMAKE_C_COMPILER emcc)
	set(CMAKE_CXX_COMPILER em++)
	set(CMAKE_TOOLCHAIN_FILE "$ENV{EMSCRIPTEN_ROOT}/cmake/Modules/Platform/Emscripten.cmake")
	set(CMAKE_AR emar)
	set(CMAKE_RANLIB emranlib)
	set(CMAKE_EXECUTABLE_SUFFIX ".js")
	list(APPEND CMAKE_PREFIX_PATH /usr/local)
	include_directories("/usr/local/include")
	include_directories("/boost_includes")

	set(EXTENDED_FLAGS " \
		--bind \
		--source-map-base ./build/ \
		--memory-init-file 0 \
		-fno-rtti \
		-DEMSCRIPTEN_HAS_UNBOUND_TYPE_NAMES=0 \
		-s NO_EXIT_RUNTIME=1 \
		-s NO_FILESYSTEM=1 \
		-s ALLOW_MEMORY_GROWTH=1 \
		-s EXPORTED_FUNCTIONS=\"['_main']\" \
		")

	if(DEFINED ENV{PSP_DEBUG})
		set(OPT_FLAGS " \
			-O0 \
			-g4 \
			-s SAFE_HEAP=1 \
			-s DISABLE_EXCEPTION_CATCHING=0 \
			-s ASSERTIONS=2 \
			-s DEMANGLE_SUPPORT=1 \
			")
	else()
		set(OPT_FLAGS " \
			-O3 \
			-g0 \
			--closure 1 \
			-s AGGRESSIVE_VARIABLE_ELIMINATION=1 \
			")
	endif()

	set(SYNC_MODE_FLAGS "-s ENVIRONMENT=node -s BINARYEN_ASYNC_COMPILATION=0 -s BINARYEN_METHOD='\"native-wasm\"' -s WASM=1")
	set(ASYNC_MODE_FLAGS "-s ENVIRONMENT=worker -s BINARYEN_ASYNC_COMPILATION=1 -s BINARYEN_METHOD='\"native-wasm\"' -s WASM=1")
	set(ASMJS_MODE_FLAGS "-s ENVIRONMENT=worker -s LEGACY_VM_SUPPORT=1 -s BINARYEN_ASYNC_COMPILATION=0 -s BINARYEN_METHOD='\"asmjs\"' -Wno-almost-asm -s WASM=0")
else()
	#####################
	# VANILLA CPP BUILD #
	#####################
	if(DEFINED ENV{PSP_DEBUG})
		set(OPT_FLAGS " \
			-O1 \
			-g3 \
			")
	else()
		set(OPT_FLAGS " \
			-O3 \
			-g0 \
			")
	endif()
	set(SYNC_MODE_FLAGS "")
	set(ASYNC_MODE_FLAGS "")
	set(ASMJS_MODE_FLAGS "")

	find_package(Boost)
	if(NOT Boost_FOUND)
		message("${Red}Boost could not be located${ColorReset}")
	else()
		message("${Cyan}Found boost in ${Boost_INCLUDE_DIRS} ${Boost_LIBRARY_DIRS} ${ColorReset}")
	endif()
	include_directories(Boost_INCLUDE_DIRS)

	find_package(TBB)
	if(NOT TBB_FOUND)
		message("${Red}TBB could not be located${ColorReset}")
	else()
		message("${Cyan}Found tbb in ${TBB_INCLUDE_DIRS} ${TBB_LIBRARY_DIRS} ${ColorReset}")
	endif()

	include_directories(TBB_INCLUDE_DIRS)
	include_directories("/usr/local/include")

	if(PSP_PYTHON_BUILD)
		#########################
		# PYTHON BINDINGS BUILD #
		#########################
		add_definitions(-DPSP_ENABLE_PYTHON)
		include_directories("/usr/local/include/python3.7m")  # FIXME
		include_directories("${CMAKE_SOURCE_DIR}/python/perspective/include")
		find_package( PythonInterp 3.7 REQUIRED )
		find_package( PythonLibs 3.7 REQUIRED )
		include_directories( ${PYTHON_INCLUDE_DIRS} )
		find_package(NumPy)

		find_package( Boost COMPONENTS python REQUIRED )
		find_library( BOOST_PYTHON boost_python)
		find_library( BOOST_NUMPY boost_numpy)
		include_directories( ${Boost_INCLUDE_DIR} )

		include_directories( ${PYTHON_NUMPY_INCLUDE_DIR})
		#####################
	endif()
endif()
#####################


set(CMAKE_C_FLAGS " \
	${CMAKE_C_FLAGS} \
	${EXTENDED_FLAGS} \
	${OPT_FLAGS} \
	")

set (SOURCE_FILES
	src/cpp/aggregate.cpp
	src/cpp/aggspec.cpp
	src/cpp/arg_sort.cpp
	src/cpp/base.cpp
	src/cpp/base_impl_linux.cpp
	src/cpp/base_impl_osx.cpp
	src/cpp/base_impl_win.cpp
	src/cpp/build_filter.cpp
	#src/cpp/calc_agg_dtype.cpp
	src/cpp/column.cpp
	src/cpp/comparators.cpp
	src/cpp/compat.cpp
	src/cpp/compat_impl_linux.cpp
	src/cpp/compat_impl_osx.cpp
	src/cpp/compat_impl_win.cpp
	src/cpp/config.cpp
	src/cpp/context_base.cpp
	src/cpp/context_grouped_pkey.cpp
	src/cpp/context_handle.cpp
	src/cpp/context_one.cpp
	src/cpp/context_two.cpp
	src/cpp/context_zero.cpp
	src/cpp/custom_column.cpp
	src/cpp/date.cpp
	src/cpp/dense_nodes.cpp
	src/cpp/dense_tree_context.cpp
	src/cpp/dense_tree.cpp
	src/cpp/dependency.cpp
	src/cpp/extract_aggregate.cpp
	src/cpp/filter.cpp
	src/cpp/flat_traversal.cpp
	src/cpp/gnode.cpp
	src/cpp/gnode_state.cpp
	src/cpp/histogram.cpp
	src/cpp/logtime.cpp
	src/cpp/mask.cpp
	src/cpp/min_max.cpp
	src/cpp/multi_sort.cpp
	src/cpp/none.cpp
	src/cpp/path.cpp
	src/cpp/pivot.cpp
	src/cpp/pool.cpp
	src/cpp/port.cpp
	src/cpp/raii.cpp
	src/cpp/raii_impl_linux.cpp
	src/cpp/raii_impl_osx.cpp
	src/cpp/raii_impl_win.cpp
	src/cpp/range.cpp
	src/cpp/rlookup.cpp
	src/cpp/scalar.cpp
	src/cpp/schema_column.cpp
	src/cpp/schema.cpp
	src/cpp/slice.cpp
	src/cpp/sort_specification.cpp
	src/cpp/sparse_tree.cpp
	src/cpp/sparse_tree_node.cpp
	src/cpp/step_delta.cpp
	src/cpp/storage.cpp
	src/cpp/storage_impl_linux.cpp
	src/cpp/storage_impl_osx.cpp
	src/cpp/storage_impl_win.cpp
	src/cpp/sym_table.cpp
	src/cpp/table.cpp
	src/cpp/time.cpp
	src/cpp/traversal.cpp
	src/cpp/traversal_nodes.cpp
	src/cpp/tree_context_common.cpp
	src/cpp/utils.cpp
	src/cpp/update_task.cpp
	src/cpp/vocab.cpp
	)

set(CMAKE_CXX_FLAGS " -std=c++0x ${CMAKE_CXX_FLAGS} ${CMAKE_C_FLAGS}")

if (PSP_WASM_BUILD)
	add_library(psp ${SOURCE_FILES})
	set_target_properties(psp PROPERTIES COMPILE_FLAGS "${ASYNC_MODE_FLAGS}")

	add_executable(perspective.async src/cpp/main.cpp)
	target_link_libraries(perspective.async psp "${ASYNC_MODE_FLAGS}")
	set_target_properties(perspective.async PROPERTIES COMPILE_FLAGS "${ASYNC_MODE_FLAGS}")
	set_target_properties(perspective.async PROPERTIES RUNTIME_OUTPUT_DIRECTORY "./build/")
	set_target_properties(perspective.async PROPERTIES OUTPUT_NAME "psp.async")

	add_executable(perspective.sync src/cpp/main.cpp)
	target_link_libraries(perspective.sync psp "${SYNC_MODE_FLAGS}")
	set_target_properties(perspective.sync PROPERTIES COMPILE_FLAGS "${SYNC_MODE_FLAGS}")
	set_target_properties(perspective.sync PROPERTIES RUNTIME_OUTPUT_DIRECTORY "./build/")
	set_target_properties(perspective.sync PROPERTIES OUTPUT_NAME "psp.sync")
	add_dependencies(perspective.sync perspective.async)

	add_executable(perspective.asm src/cpp/main.cpp)
	target_link_libraries(perspective.asm psp "${ASMJS_MODE_FLAGS}")
	set_target_properties(perspective.asm PROPERTIES COMPILE_FLAGS "${ASMJS_MODE_FLAGS}")
	set_target_properties(perspective.asm PROPERTIES RUNTIME_OUTPUT_DIRECTORY "./build/")
	set_target_properties(perspective.asm PROPERTIES OUTPUT_NAME "psp.asmjs")
	add_dependencies(perspective.asm perspective.sync)
else()
	if(PSP_PYTHON_BUILD)
		########################
		# Python extra targets #
		########################
		add_library(psp SHARED ${SOURCE_FILES} ${CMAKE_SOURCE_DIR}/python/perspective/src/numpy.cpp ${HEADER_FILES})
		target_link_libraries(psp ${BOOST_PYTHON})
		target_link_libraries(psp ${BOOST_NUMPY})
		target_link_libraries(psp ${Boost_LIBRARIES} ${PYTHON_LIBRARIES})
		set(CMAKE_SHARED_LIBRARY_SUFFIX .so)

		add_library(binding SHARED ${CMAKE_SOURCE_DIR}/python/perspective/src/binding.cpp)
		target_link_libraries(binding psp)
		target_link_libraries(binding tbb)
		target_link_libraries(binding ${BOOST_PYTHON})
		target_link_libraries(binding ${BOOST_NUMPY})
		target_link_libraries(binding ${Boost_LIBRARIES} ${PYTHON_LIBRARIES})
		########################
	else()
		add_library(psp SHARED ${SOURCE_FILES} ${HEADER_FILES})
		if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
			set(CMAKE_SHARED_LIBRARY_SUFFIX dylib)
		endif()
	endif()

	if(PSP_CPP_BUILD_STRICT)
		target_compile_options(psp PRIVATE -Wall -Werror)
	endif()

	# target_link_libraries(psp ${TBB_LIBRARIES}) // doesnt work on mac
	target_link_libraries(psp tbb)
	set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} ${TBB_LIBRARY})

	target_compile_options(psp PRIVATE $<$<CONFIG:DEBUG>:-fPIC -O0>)
endif()


##############################
# helper to grab gtest et al #
##############################
function (psp_build_dep name cmake_file)
	if(EXISTS ${CMAKE_BINARY_DIR}/${name}-build)
		message(WARNING "${Cyan}Dependency found - not rebuilding - ${CMAKE_BINARY_DIR}/${name}-build${ColorReset}")

	else()
		configure_file(${cmake_file} ${name}-download/CMakeLists.txt)
		execute_process(COMMAND ${CMAKE_COMMAND} -G "${CMAKE_GENERATOR}" .
			RESULT_VARIABLE result
			WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/${name}-download )
		if(result)
			message(FATAL_ERROR "CMake step for ${name} failed: ${result}")
		endif()
		execute_process(COMMAND ${CMAKE_COMMAND} --build .
			RESULT_VARIABLE result
			WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/${name}-download )
		if(result)
			message(FATAL_ERROR "Build step for ${name} failed: ${result}")
		endif()
	endif()
	add_subdirectory(${CMAKE_BINARY_DIR}/${name}-src
		${CMAKE_BINARY_DIR}/${name}-build
		EXCLUDE_FROM_ALL)
	include_directories(${CMAKE_BINARY_DIR}/${name}-src/extras/gtest/include)
	include_directories(${CMAKE_BINARY_DIR}/${name}-src/include)
	include_directories(${CMAKE_BINARY_DIR}/${name}-src)
endfunction()
##############################


#############
# CPP TESTS #
#############
if (PSP_CPP_BUILD_TESTS)
	if (WIN32)
		set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
	endif()
	psp_build_dep("googletest" "cmake/GTest.txt.in")
	# psp_build_dep("tbb" "cmake/TBB.txt.in")

	add_subdirectory(test)
	add_custom_target(gcov
		COMMAND mkdir -p ${CMAKE_BINARY_DIR}/coverage
		WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
	)
	add_custom_command(TARGET gcov
		COMMAND echo "=================== GCOV ===================="
		COMMAND gcovr -r ../ --html --html-details -o coverage/index.html
		COMMAND xdg-open coverage/index.html
		WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
	)
endif()
#############
