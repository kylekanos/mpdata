# getting a list of tests
execute_process(
  COMMAND "find" "${CMAKE_SOURCE_DIR}/tests" "-name" "*:in"
  COMMAND "cut" "-d" ":" "-f" "1"
  COMMAND "awk" "{printf(\"%s;\",$1)}" # to make it a CMake list
  OUTPUT_VARIABLE TESTS
)

add_test(build_test_for "${CMAKE_COMMAND}" --build ${CMAKE_BINARY_DIR} --target test-for)
add_test(build_test_cpp "${CMAKE_COMMAND}" --build ${CMAKE_BINARY_DIR} --target test-cpp)

# registering the tests to be run by CTest 
foreach(TEST ${TESTS})
  string(STRIP ${TEST} TEST)
  get_filename_component(NAME "${TEST}" NAME)

  # parsing filenames
  string(REGEX MATCHALL "nx=(.+)_ny=(.+)_Cx=(.+)_Cy=(.+)_nt=(.+)_it=(.+)" NX ${NAME})
  set(NX ${CMAKE_MATCH_1}) # \__ domain size
  set(NY ${CMAKE_MATCH_2}) # /
  set(CX ${CMAKE_MATCH_3}) # \__ Courant numbers
  set(CY ${CMAKE_MATCH_4}) # /
  set(NT ${CMAKE_MATCH_5}) # --- number of timesteps
  set(IT ${CMAKE_MATCH_6}) # --- number of MPDATA iterations
  set(DEC 5) # ----------------- desired precision (abs(desired-actual) < 0.5 * 10**(-decimal))

  # passing all of the above as command-line arguments to the test programs
  set(ARG ${NX};${NY};${CX};${CY};${NT};${IT};${NAME}:in;${NAME}:out;${DEC})

  # C++
  add_test(NAME "cpp:${NAME}" COMMAND ${CMAKE_BINARY_DIR}/cpp/test-cpp ${ARG})
  set_tests_properties("cpp:${NAME}" PROPERTIES DEPENDS build_test_cpp)
  
  # Python
  add_test(NAME "pyt:${NAME}" COMMAND python ${CMAKE_BINARY_DIR}/pyt/test.py ${ARG})

  # PyPy
  add_test(NAME "pyp:${NAME}" COMMAND pypy-c1.9 ${CMAKE_BINARY_DIR}/pyt/test.py ${ARG})

  # Fortran
  add_test(NAME "for:${NAME}" COMMAND ${CMAKE_BINARY_DIR}/for/test-for ${ARG})
  set_tests_properties("for:${NAME}" PROPERTIES DEPENDS build_test_for)
endforeach()