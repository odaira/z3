################################################################################
# API header files
################################################################################
# This lists the API header files that are scanned by
# some of the build rules to generate some files needed
# by the build
set(Z3_API_HEADER_FILES_TO_SCAN
  z3_api.h
  z3_ast_containers.h
  z3_algebraic.h
  z3_polynomial.h
  z3_rcf.h
  z3_fixedpoint.h
  z3_optimization.h
  z3_interp.h
  z3_fpa.h
)
set(Z3_FULL_PATH_API_HEADER_FILES_TO_SCAN "")
foreach (header_file ${Z3_API_HEADER_FILES_TO_SCAN})
  set(full_path_api_header_file "${CMAKE_CURRENT_SOURCE_DIR}/api/${header_file}")
  list(APPEND Z3_FULL_PATH_API_HEADER_FILES_TO_SCAN "${full_path_api_header_file}")
  if (NOT EXISTS "${full_path_api_header_file}")
    message(FATAL_ERROR "API header file \"${full_path_api_header_file}\" does not exist")
  endif()
endforeach()

################################################################################
# Traverse directories each adding a Z3 component
################################################################################
# I'm duplicating the order in ``mk_project.py`` for now to help us keep
# the build systems in sync.
#
# The components in these directory explicitly declare their dependencies so
# you may be able to re-order some of these directories but an error will be
# raised if you try to declare a component is dependent on another component
# that has not yet been declared.
add_subdirectory(util)
add_subdirectory(math/polynomial)
add_subdirectory(sat)
add_subdirectory(nlsat)
add_subdirectory(math/hilbert)
add_subdirectory(math/simplex)
add_subdirectory(math/automata)
add_subdirectory(math/interval)
add_subdirectory(math/realclosure)
add_subdirectory(math/subpaving)
add_subdirectory(ast)
add_subdirectory(ast/rewriter)
add_subdirectory(ast/normal_forms)
add_subdirectory(model)
add_subdirectory(tactic)
add_subdirectory(ast/substitution)
add_subdirectory(parsers/util)
add_subdirectory(math/grobner)
add_subdirectory(math/euclid)
add_subdirectory(tactic/core)
add_subdirectory(sat/tactic)
add_subdirectory(tactic/arith)
add_subdirectory(nlsat/tactic)
add_subdirectory(math/subpaving/tactic)
add_subdirectory(tactic/aig)
add_subdirectory(solver)
add_subdirectory(ackermannization)
add_subdirectory(interp)
add_subdirectory(cmd_context)
add_subdirectory(cmd_context/extra_cmds)
add_subdirectory(parsers/smt2)
add_subdirectory(ast/proof_checker)
## Simplifier module will be deleted in the future.
## It has been replaced with rewriter component.
add_subdirectory(ast/simplifier)
add_subdirectory(ast/fpa)
add_subdirectory(ast/macros)
add_subdirectory(ast/pattern)
add_subdirectory(ast/rewriter/bit_blaster)
add_subdirectory(smt/params)
add_subdirectory(smt/proto_model)
add_subdirectory(smt)
add_subdirectory(tactic/bv)
add_subdirectory(smt/tactic)
add_subdirectory(tactic/sls)
add_subdirectory(qe)
add_subdirectory(duality)
add_subdirectory(muz/base)
add_subdirectory(muz/dataflow)
add_subdirectory(muz/transforms)
add_subdirectory(muz/rel)
add_subdirectory(muz/pdr)
add_subdirectory(muz/clp)
add_subdirectory(muz/tab)
add_subdirectory(muz/bmc)
add_subdirectory(muz/ddnf)
add_subdirectory(muz/duality)
add_subdirectory(muz/fp)
add_subdirectory(tactic/nlsat_smt)
add_subdirectory(tactic/ufbv)
add_subdirectory(sat/sat_solver)
add_subdirectory(tactic/smtlogics)
add_subdirectory(tactic/fpa)
add_subdirectory(tactic/portfolio)
add_subdirectory(parsers/smt)
add_subdirectory(opt)
add_subdirectory(api)
add_subdirectory(api/dll)

################################################################################
# libz3
################################################################################
get_property(Z3_LIBZ3_COMPONENTS_LIST GLOBAL PROPERTY Z3_LIBZ3_COMPONENTS)
set (object_files "")
foreach (component ${Z3_LIBZ3_COMPONENTS_LIST})
  list(APPEND object_files $<TARGET_OBJECTS:${component}>)
endforeach()
if (BUILD_LIBZ3_SHARED)
  set(lib_type "SHARED")
else()
  set(lib_type "STATIC")
endif()
add_library(libz3 ${lib_type} ${object_files})
# FIXME: Set "VERSION" and "SOVERSION" properly
set_target_properties(libz3 PROPERTIES
  # FIXME: Should we be using ${Z3_VERSION} here?
  # VERSION: Sets up symlinks, does it do anything else?
  VERSION ${Z3_VERSION}
  # SOVERSION: On platforms that use ELF this sets the API version
  # and should be incremented everytime the API changes
  SOVERSION ${Z3_VERSION})

if (NOT MSVC)
  # On UNIX like platforms if we don't change the OUTPUT_NAME
  # the library gets a name like ``liblibz3.so`` so we change it
  # here. We don't do a rename with MSVC because we get file naming
  # conflicts (the z3 executable also has this OUTPUT_NAME) with
  # ``.ilk``, ``.pdb``, ``.lib`` and ``.exp`` files sharing the same
  # prefix.
  set_target_properties(libz3 PROPERTIES OUTPUT_NAME z3)
endif()

# Using INTERFACE means that targets that try link against libz3 will
# automatically link against the libs in Z3_DEPENDENT_LIBS
target_link_libraries(libz3 INTERFACE ${Z3_DEPENDENT_LIBS})

z3_append_linker_flag_list_to_target(libz3 ${Z3_DEPENDENT_EXTRA_CXX_LINK_FLAGS})

# Declare which header file are the public header files of libz3
# these will automatically installed when the libz3 target is installed
set (libz3_public_headers
  z3_algebraic.h
  z3_api.h
  z3_ast_containers.h
  z3_fixedpoint.h
  z3_fpa.h
  z3.h
  c++/z3++.h
  z3_interp.h
  z3_macros.h
  z3_optimization.h
  z3_polynomial.h
  z3_rcf.h
  z3_v1.h
)
foreach (header ${libz3_public_headers})
  set_property(TARGET libz3 APPEND PROPERTY
    PUBLIC_HEADER "${CMAKE_SOURCE_DIR}/src/api/${header}")
endforeach()

install(TARGETS libz3
  LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
  ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
  PUBLIC_HEADER DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

if (MSVC)
  # Handle settings dll exports when using MSVC
  # FIXME: This seems unnecessarily complicated but I'm doing
  # this because this is what the python build system does.
  # CMake has a much more elegant (see ``GenerateExportHeader.cmake``)
  # way of handling this.
  set(dll_module_exports_file "${CMAKE_CURRENT_BINARY_DIR}/api_dll.def")
  add_custom_command(OUTPUT "${dll_module_exports_file}"
    COMMAND
      "${PYTHON_EXECUTABLE}"
      "${CMAKE_SOURCE_DIR}/scripts/mk_def_file.py"
      "${dll_module_exports_file}"
      "libz3"
      ${Z3_FULL_PATH_API_HEADER_FILES_TO_SCAN}
    DEPENDS
      "${CMAKE_SOURCE_DIR}/scripts/mk_def_file.py"
      ${Z3_GENERATED_FILE_EXTRA_DEPENDENCIES}
      ${Z3_FULL_PATH_API_HEADER_FILES_TO_SCAN}
    COMMENT "Generating \"${dll_module_exports_file}\""
    ${ADD_CUSTOM_COMMAND_USES_TERMINAL_ARG}
    VERBATIM
  )
  add_custom_target(libz3_extra_depends
    DEPENDS "${dll_module_exports_file}"
  )
  add_dependencies(libz3 libz3_extra_depends)
  z3_append_linker_flag_list_to_target(libz3 "/DEF:${dll_module_exports_file}")
endif()

################################################################################
# Z3 executable
################################################################################
add_subdirectory(shell)

################################################################################
# z3-test
################################################################################
add_subdirectory(test)


################################################################################
# Z3 API bindings
################################################################################
option(BUILD_PYTHON_BINDINGS "Build Python bindings for Z3" OFF)
if (BUILD_PYTHON_BINDINGS)
  if (NOT BUILD_LIBZ3_SHARED)
    message(FATAL_ERROR "The python bindings will not work with a static libz3. "
            "You either need to disable BUILD_PYTHON_BINDINGS or enable BUILD_LIBZ3_SHARED")
  endif()
  add_subdirectory(api/python)
endif()

# TODO: Implement support for other bindigns
