if(ENABLE_DATA_TOOLS)
  file(GLOB_RECURSE TEST_FILES "${CMAKE_CURRENT_LIST_DIR}/test_*.cc")

  add_custom_target(gurka)
  set_target_properties(gurka PROPERTIES FOLDER "Tests")
  add_custom_target(run-gurka)
  set_target_properties(run-gurka PROPERTIES FOLDER "Tests")


  ## Add executable targets
  foreach(FULLFILENAME IN ITEMS ${TEST_FILES})
    file(RELATIVE_PATH FILENAME "${CMAKE_CURRENT_LIST_DIR}" ${FULLFILENAME})
    string(REGEX REPLACE "test_(.*).cc" "gurka_\\1" TESTNAME ${FILENAME})
    add_executable(${TESTNAME} EXCLUDE_FROM_ALL ${FILENAME} ${VALHALLA_SOURCE_DIR}/third_party/microtar/src/microtar.c)
    set_target_properties(${TESTNAME} PROPERTIES
      FOLDER "Tests"
      COMPILE_DEFINITIONS VALHALLA_SOURCE_DIR="${VALHALLA_SOURCE_DIR}/")
    create_source_groups("Source Files" ${FILENAME} ${VALHALLA_SOURCE_DIR}/third_party/microtar/src/microtar.c)
    target_link_libraries(${TESTNAME} valhalla gtest_main gmock pthread)
    if (LUAJIT_FOUND AND APPLE)
      # Using LuaJIT on macOS requires a couple of extra linker flags
      target_link_options(${TESTNAME} PUBLIC -pagezero_size 10000 -image_base 100000000)
    endif()
    target_include_directories(${TESTNAME} SYSTEM PRIVATE
      ${VALHALLA_SOURCE_DIR}/third_party/microtar/src
      ${VALHALLA_SOURCE_DIR}/third_party/protozero/include
      ${VALHALLA_SOURCE_DIR}/third_party/libosmium/include)
    add_dependencies(${TESTNAME} build_timezones)

    ## Test run targets
    add_custom_command(OUTPUT ${TESTNAME}.log
      COMMAND
        LOCPATH=${VALHALLA_SOURCE_DIR}/locales
        /bin/bash -c "${CMAKE_CURRENT_BINARY_DIR}/${TESTNAME} >& ${CMAKE_CURRENT_BINARY_DIR}/${TESTNAME}.log \
        && echo [PASS] ${TESTNAME} \
        || ( exit=$? ; \
             echo [FAIL] ${TESTNAME} ; \
             cat ${CMAKE_CURRENT_BINARY_DIR}/${TESTNAME}.log ; \
             exit $exit )"
      WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
      DEPENDS ${TESTNAME}
      VERBATIM)
    add_custom_target(run-${TESTNAME} DEPENDS ${TESTNAME}.log)
    set_target_properties(run-${TESTNAME} PROPERTIES FOLDER "Tests")
    add_dependencies(gurka ${TESTNAME})
    add_dependencies(run-gurka run-${TESTNAME})
  endforeach()

  add_dependencies(tests gurka)
  add_dependencies(check run-gurka)
endif()
