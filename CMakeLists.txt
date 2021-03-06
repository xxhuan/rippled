# !!! The official build system is SConstruct !!!
# This is an experimental cmake build file for rippled
#
# cmake support in rippled. Currently supports:
#
#  * unity/nounity debug/release
#  * running protobuf
#  * sanitizer builds
#  * optional release build with assert turned on
#  * `target` variable to easily set compiler/debug/unity
#     (i.e. -Dtarget=gcc.debug.nounity)
#  * gcc/clang/visual studio/xcode
#  * linux/mac/win
#  * gcc 4 ABI, when needed
#  * ninja builds
#  * check openssl version on linux
#  * static builds (swd TBD: needs to be tested by building & deploying on different systems)
#
# TBD:
#  * jemalloc support
#  * count
#  * Windows protobuf compiler puts generated file in src directory instead of build directory.
#
# Notes:
#  * Use the -G"Visual Studio 14 2015 Win64" generator on Windows. Without this
#    a 32-bit project will be created. There is no way to set the generator or
#    force a 64-bit build in CMakeLists.txt (setting CMAKE_GENERATOR_PLATFORM won't work).
#    The best solution may be to wrap cmake with a script.
#
#  * It is not possible to generate a visual studio project on linux or
#    mac. The visual studio generator is only available on windows.
#
#  * The visual studio project can be _either_ unity or
#    non-unity (selected at generation time).  It does not appear possible
#    to disable compilation based on configuration.
#
#  * Language is _much_ worse than python, poor documentation and "quirky"
#    language support (for example, generator expressions can only be used
#    in limited contexts and seem to work differently based on
#    context (set_property can set multiple values, add_compile_options
#    can not/or is buggy)
#
#  * Could not call out to `sed` because cmake messed with the regular
#    expression before calling the external command. I did not see a way
#    around this.
#
#  * Makefile generators want to be single target. It wants a separate
#    directory for each target type. I saw some mentions on the web for
#    ways around this bug haven't look into it. The visual studio project
#    does support debug/release configurations in the same project (but
#    not unity/non-unity).

############################################################

cmake_minimum_required(VERSION 3.1.0)

if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
  set(dir "build")
  set(cmd "cmake")
  if (target)
    set(dir "${dir}/${target}")
    set(cmd "${cmd} -Dtarget=${target}")
  elseif(CMAKE_BUILD_TYPE)
    set(dir "${dir}/${CMAKE_BUILD_TYPE}")
    set(cmd "${cmd} -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}")
  else()
    set(dir "${dir}/default")
  endif()
  set(cmd "${cmd} ${CMAKE_SOURCE_DIR}")

  message(FATAL_ERROR "Builds are not allowed in ${CMAKE_SOURCE_DIR}.\n"
    "Instead:\n"
    "1) Remove the CMakeCache.txt file and CMakeFiles directory "
   "from ${CMAKE_SOURCE_DIR}.\n"
    "2) Create a directory to hold your build files, for example: ${dir}.\n"
    "3) Change to that directory.\n"
    "4) Run cmake targetting ${CMAKE_SOURCE_DIR}, for example: ${cmd}")
endif()
if("${CMAKE_GENERATOR}" MATCHES "Visual Studio" AND
    NOT ("${CMAKE_GENERATOR}" MATCHES .*Win64.*))
  message(FATAL_ERROR "Visual Studio 32-bit build is unsupported. Use
    -G\"${CMAKE_GENERATOR} Win64\"")
endif()

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/Builds/CMake")
include(CMakeFuncs)

set(openssl_min 1.0.2)

parse_target()

if (NOT DEFINED unity)
  set(unity true)
  set(nonunity false)
endif()

setup_build_cache()

project(rippled)

if(nonunity)
  set(CMAKE_CXX_FLAGS_DEBUGCLASSIC ${CMAKE_CXX_FLAGS_DEBUG})
  set(CMAKE_CXX_FLAGS_RELEASECLASSIC ${CMAKE_CXX_FLAGS_RELEASE})
  set(CMAKE_EXE_LINKER_FLAGS_DEBUGCLASSIC
    ${CMAKE_EXE_LINKER_FLAGS_DEBUG})
  set(CMAKE_EXE_LINKER_FLAGS_RELEASECLASSIC
    ${CMAKE_EXE_LINKER_FLAGS_RELEASE})
endif()

determine_build_type()

check_gcc4_abi()

############################################################

include_directories(
  src
  src/beast
  src/beast/include
  src/beast/extras
  src/nudb/include
  src/soci/src
  src/soci/include)

special_build_flags()

############################################################

find_boost(
  coroutine
  context
  date_time
  filesystem
  program_options
  regex
  system
  thread)

find_pthread()

find_openssl(${openssl_min})

find_protobuf()

setup_build_boilerplate()

############################################################

if (is_clang)
  set(rocks_db_system_header --system-header-prefix=rocksdb2)
else()
  unset(rocks_db_system_header)
endif()

set(soci_extra_includes
  -I"${CMAKE_SOURCE_DIR}/"src/soci/src/core
  -I"${CMAKE_SOURCE_DIR}/"src/soci/include/private
  -I"${CMAKE_SOURCE_DIR}/"src/sqlite)

if (WIN32 OR is_xcode OR unity)
  prepend(beast_unity_srcs
    src/ripple/beast/unity/
    beast_insight_unity.cpp
    beast_net_unity.cpp
    beast_utility_unity.cpp)

  prepend(ripple_unity_srcs
    src/ripple/unity/
    app_ledger.cpp
    app_main.cpp
    app_misc.cpp
    app_paths.cpp
    app_tx.cpp
    core.cpp
    basics.cpp
    crypto.cpp
    ledger.cpp
    net.cpp
    overlay.cpp
    peerfinder.cpp
    json.cpp
    protocol.cpp
    rpcx.cpp
    shamap.cpp
    server.cpp
    test.cpp)
  
  prepend(test_unity_srcs 
    src/unity/
    app_test_unity.cpp
    basics_test_unity.cpp
    beast_test_unity.cpp
    core_test_unity.cpp
    json_test_unity.cpp
    ledger_test_unity.cpp
    overlay_test_unity.cpp
    peerfinder_test_unity.cpp
    protocol_test_unity.cpp
    resource_test_unity.cpp
    rpc_test_unity.cpp
    server_test_unity.cpp
    shamap_test_unity.cpp
    test_unity.cpp)
    
  list(APPEND rippled_src ${beast_unity_srcs} ${ripple_unity_srcs} ${test_unity_srcs})
 
  

  add_with_props(rippled_src src/unity/nodestore_test_unity.cpp
    -I"${CMAKE_SOURCE_DIR}/"src/rocksdb2/include
    -I"${CMAKE_SOURCE_DIR}/"src/snappy/snappy
    -I"${CMAKE_SOURCE_DIR}/"src/snappy/config
    ${rocks_db_system_header})
  
  add_with_props(rippled_src src/ripple/unity/nodestore.cpp
    -I"${CMAKE_SOURCE_DIR}/"src/rocksdb2/include
    -I"${CMAKE_SOURCE_DIR}/"src/snappy/snappy
    -I"${CMAKE_SOURCE_DIR}/"src/snappy/config
    ${rocks_db_system_header})

  add_with_props(rippled_src src/ripple/unity/soci_ripple.cpp ${soci_extra_includes})

  list(APPEND ripple_unity_srcs ${beast_unity_srcs} ${test_unity_srcs}
    src/ripple/unity/nodestore.cpp
    src/ripple/unity/soci_ripple.cpp
    src/unity/nodestore_test_unity.cpp)

  set_property(
    SOURCE ${ripple_unity_srcs}
    APPEND
    PROPERTY HEADER_FILE_ONLY
    ${nonunity})
  # Doesn't work
  # $<OR:$<CONFIG:DebugClassic>,$<CONFIG:ReleaseClassic>>)

endif ()

if (WIN32 OR is_xcode OR NOT unity)
  # Rippled
  file(GLOB_RECURSE core_srcs src/ripple/core/*.cpp)
  add_with_props(rippled_src "${core_srcs}"
    -I"${CMAKE_SOURCE_DIR}/"src/soci/src/core
    -I"${CMAKE_SOURCE_DIR}/"src/sqlite)

  set(non_unity_srcs ${core_srcs})

  foreach(curdir
      beast/clock
      beast/container
      beast/insight
      beast/net
      beast/utility
      app
      basics
      crypto
      json
      ledger
      legacy
      net
      overlay
      peerfinder
      protocol
      rpc
      server
      shamap
      test)
    file(GLOB_RECURSE cursrcs src/ripple/${curdir}/*.cpp)
    list(APPEND rippled_src "${cursrcs}")
    list(APPEND non_unity_srcs "${cursrcs}")
  endforeach()

  file(GLOB_RECURSE nodestore_srcs src/ripple/nodestore/*.cpp
    src/test/nodestore/*.cpp)

  add_with_props(rippled_src "${nodestore_srcs}"
    -I"${CMAKE_SOURCE_DIR}/"src/rocksdb2/include
    -I"${CMAKE_SOURCE_DIR}/"src/snappy/snappy
    -I"${CMAKE_SOURCE_DIR}/"src/snappy/config
    ${rocks_db_system_header})
  
  list(APPEND non_unity_srcs "${nodestore_srcs}")
  

  file(GLOB_RECURSE test_srcs src/test/*.cpp)
  add_with_props("${test_srcs}"
    -I"${CMAKE_SOURCE_DIR}/"src/rocksdb2/include
    -I"${CMAKE_SOURCE_DIR}/"src/snappy/snappy
    -I"${CMAKE_SOURCE_DIR}/"src/snappy/config
    ${rocks_db_system_header})
  
  list(APPEND non_unity_srcs "${test_srcs}")
  list(APPEND rippled_src "${test_srcs}")

  file(GLOB_RECURSE rippled_headers src/*.h src/*.hpp)
  foreach(curdir
      beast/asio
      beast/core
      beast/crypto
      beast/cxx17
      beast/hash
      proto
      resource
      validators
      websocket)
    file(GLOB_RECURSE cursrcs src/ripple/${curdir}/*.cpp)
    list(APPEND rippled_headers "${cursrcs}")
  endforeach()
  list(APPEND rippled_src "${rippled_headers}")

  # Properties
  set_property(
    SOURCE ${non_unity_srcs}
    APPEND
    PROPERTY HEADER_FILE_ONLY
    ${unity})
  set_property(
    SOURCE ${rippled_headers}
    APPEND
    PROPERTY HEADER_FILE_ONLY
    true)
    # Doesn't work
    # $<OR:$<CONFIG:Debug>,$<CONFIG:Release>>)
endif()

if (WIN32 OR is_xcode)
  # Documentation sources
  prepend(doc_srcs
    docs/
    Jamfile.v2
    boostbook.dtd
    index.xml
    main.qbk
    quickref.xml
    reference.xsl
    source.dox)

  set_property(
    SOURCE ${doc_srcs}
    APPEND
    PROPERTY HEADER_FILE_ONLY
    true)
    # Doesn't work
    # $<OR:$<CONFIG:Debug>,$<CONFIG:Release>>)
endif()

############################################################

add_with_props(rippled_src src/ripple/unity/soci.cpp
  ${soci_extra_includes})

if (NOT is_msvc)
  set(no_unused_w -Wno-unused-function)
else()
  unset(no_unused_w)
endif()

add_with_props(rippled_src src/ripple/unity/secp256k1.cpp
  -I"${CMAKE_SOURCE_DIR}/"src/secp256k1
  ${no_unused_w}
  )

foreach(cursrc
    src/ripple/beast/unity/beast_hash_unity.cpp
    src/ripple/unity/beast.cpp
    src/ripple/unity/lz4.c
    src/ripple/unity/protobuf.cpp
    src/ripple/unity/ripple.proto.cpp
    src/ripple/unity/resource.cpp
    src/ripple/unity/websocket02.cpp)

  add_with_props(rippled_src ${cursrc}
    ${rocks_db_system_header}
    )

endforeach()

if (NOT is_msvc)
  set(extra_props -Wno-array-bounds)
else()
  unset(extra_props)
endif()

add_with_props(rippled_src src/sqlite/sqlite_unity.c
  ${extra_props})

if (NOT is_msvc)
  set_source_files_properties(src/ripple/unity/beastc.c
    PROPERTIES COMPILE_FLAGS
    -Wno-array-bounds)
endif()

add_with_props(rippled_src src/ripple/unity/ed25519.c
  -I"${CMAKE_SOURCE_DIR}/"src/ed25519-donna)

if (is_gcc)
  set(no_init_w -Wno-maybe-uninitialized)
else()
  unset(no_init_w)
endif()

add_with_props(rippled_src src/ripple/unity/rocksdb.cpp
  -I"${CMAKE_SOURCE_DIR}/"src/rocksdb2
  -I"${CMAKE_SOURCE_DIR}/"src/rocksdb2/include
  -I"${CMAKE_SOURCE_DIR}/"src/snappy/snappy
  -I"${CMAKE_SOURCE_DIR}/"src/snappy/config
  ${no_init_w} ${rocks_db_system_header})

if (NOT is_msvc)
  set(no_unused_w -Wno-unused-function)
endif()

add_with_props(rippled_src src/ripple/unity/snappy.cpp
  -I"${CMAKE_SOURCE_DIR}/"src/snappy/snappy
  -I"${CMAKE_SOURCE_DIR}/"src/snappy/config
  ${no_unused_w})

if (APPLE AND is_clang)
  list(APPEND rippled_src src/ripple/unity/beastobjc.mm)
endif()

############################################################

if (WIN32 OR is_xcode)
  group_sources(src)
  group_sources(docs)
endif()

add_executable(rippled ${rippled_src} ${PROTO_HDRS})

find_program(
  B2_EXE
  NAMES b2
  PATHS ENV BOOST_ROOT
  DOC "Location of the b2 build executable from Boost")
if(${B2_EXE} STREQUAL "b2-NOTFOUND")
  message(WARNING
    "Boost b2 executable not found. docs target will not be buildable")
endif()
add_custom_target(docs
  COMMAND "./makeqbk.sh"
  COMMAND ${B2_EXE}
  BYPRODUCTS "${CMAKE_SOURCE_DIR}/docs/html/index.html"
  WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/docs"
  SOURCES "${doc_srcs}"
  )

create_build_folder(rippled)

set_startup_project(rippled)

target_link_libraries(rippled
  ${OPENSSL_LIBRARIES} ${PROTOBUF_LIBRARIES} ${SANITIZER_LIBRARIES})

link_common_libraries(rippled)

if (NOT CMAKE_SIZEOF_VOID_P EQUAL 8)
  message(WARNING "Rippled requires a 64 bit target architecture.\n"
    "The most likely cause of this warning is trying to build rippled with a 32-bit OS.")
endif()
