# YAEP (Yet Another Earley Parser)
#
# Copyright (c) 1997-2018  Vladimir Makarov <vmakarov@gcc.gnu.org>
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the
# "Software"), to deal in the Software without restriction, including
# without limitation the rights to use, copy, modify, merge, publish,
# distribute, sublicense, and/or sell copies of the Software, and to
# permit persons to whom the Software is furnished to do so, subject to
# the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
# OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
# MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
# IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
# CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
# TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
# SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

# FIXME: benchmark needs input redirection, should be fixed
find_program( SHELL bash )
if( SHELL STREQUAL "SHELL-NOTFOUND" )
	message( WARNING "Cannot run comparison test without bash" )
	return()
endif()

# We need token generation from (f)lex for all tests
if( NOT FLEX_FOUND )
	return()
endif()

include_directories(
	${CMAKE_CURRENT_SOURCE_DIR}
	${PROJECT_SOURCE_DIR}/src
	${PROJECT_SOURCE_DIR}/test
	${PROJECT_BINARY_DIR}/test
	${CMAKE_CURRENT_BINARY_DIR}
)
link_directories(
	${PROJECT_BINARY_DIR}/src
)

# Prepare test files
set( BIGTEST_DATA "${CMAKE_CURRENT_SOURCE_DIR}/whole_gcc_test.i" )
set( TEST_DATA "${CMAKE_CURRENT_BINARY_DIR}/test.i" )
configure_file( test.i "${TEST_DATA}" COPYONLY )
file( READ test1.i TEST1 )
foreach( i RANGE 0 9 )
	file( APPEND "${TEST_DATA}" "${TEST1}" )
endforeach()

# Check for test code generators
find_package( BISON )
find_program( MSTA msta )
if( NOT ( MSTA STREQUAL "MSTA-NOTFOUND" ) )
	set( MSTA_FOUND TRUE )
endif()

# Check for timing program
find_program( TIME time )
if( TIME STREQUAL "TIME-NOTFOUND" )
	set( TIMER "" )
else()
	set( TIMER "${TIME}" )
endif()

# Run tests with bison
if( BISON_FOUND )
	bison_target( bison_ansic ansic.y "${CMAKE_CURRENT_BINARY_DIR}/y.tab.c" )
	add_custom_target( bison_ansic_output DEPENDS y.tab.h y.tab.c )
	add_executable( ansic-bison test_yyparse.c )
	add_dependencies( ansic-bison ansic_c bison_ansic_output )
	target_link_libraries( ansic-bison
		ticker
		yaep
	)
	add_test( ansic-test-bison-small ${SHELL} -c "${TIMER} ${CMAKE_CURRENT_BINARY_DIR}/ansic-bison < ${TEST_DATA}" )
	add_test( ansic-test-bison-big ${SHELL} -c "${TIMER} ${CMAKE_CURRENT_BINARY_DIR}/ansic-bison < ${BIGTEST_DATA}" )
else()
	message( WARNING "bison/yacc not found; not performing ANSI-C tests with bison" )
endif()

# Run tests with msta
if( MSTA_FOUND )
	add_custom_command( OUTPUT y.tab.h y.tab.c COMMAND ${MSTA} -yacc-file-names ansic.y ) # FIXME same output as bison above
	add_custom_target( msta_bison_output DEPENDS y.tab.h y.tab.c )
	add_executable( ansic-msta test_yyparse.c )
	add_dependencies( ansic-msta ansic_c msta_bison_output )
	target_link_libraries( ansic-msta
		ticker
		yaep
	)
	add_test( ansic-test-msta-small ${SHELL} -c "${TIMER} ${CMAKE_CURRENT_BINARY_DIR}/ansic-msta < ${TEST_DATA}" )
	add_test( ansic-test-msta-big ${SHELL} -c "${TIMER} ${CMAKE_CURRENT_BINARY_DIR}/ansic-msta < ${BIGTEST_DATA}" )
else()
	message( WARNING "MSTA not found; not performing ANSI-C tests with MSTA" )
endif()

# Pure YAEP tests
add_executable( ansic-yaep test_yaep.c )
add_dependencies( ansic-yaep ansic_c )
target_link_libraries( ansic-yaep
	ticker
	yaep
)
add_test( ansic-test-yaep-no-lookahead ${SHELL} -c "${TIMER} ${CMAKE_CURRENT_BINARY_DIR}/ansic-yaep 0 0 < ${TEST_DATA}" )
add_test( ansic-test-yaep-static-lookahead ${SHELL} -c "${TIMER} ${CMAKE_CURRENT_BINARY_DIR}/ansic-yaep 1 0 < ${TEST_DATA}" )
add_test( ansic-test-yaep-dynamic-lookahead ${SHELL} -c "${TIMER} ${CMAKE_CURRENT_BINARY_DIR}/ansic-yaep 2 0 < ${TEST_DATA}" )
add_test( ansic-test-yaep-big ${SHELL} -c "${TIMER} ${CMAKE_CURRENT_BINARY_DIR}/ansic-yaep 1 0 < ${BIGTEST_DATA}" )

# LOTSAWA tests
find_path( LOTSAWA_INCLUDE LotsawaC.h )
if ( LOTSAWA_INCLUDE STREQUAL "LOTSAWA_INCLUDE-NOTFOUND" )
	message( WARNING "LOTSAWA header not found; not performing LOTSAWA ANSI-C test" )
	return()
endif()
include_directories(
	${LOTSAWA_INCLUDE}
)
find_library( LOTSAWA_LIBS NAMES LotsawaC )
if( LOTSAWA_LIBS STREQUAL "LOTSAWA_LIBS-NOTFOUND" )
	message( WARNING "LOTSAWA library not found; not performing LOTSAWA ANSI-C test" )
	return()
endif()
# MARPA tests
find_path( MARPA_INCLUDE marpa.h )
if ( MARPA_INCLUDE STREQUAL "MARPA_INCLUDE-NOTFOUND" )
	message( WARNING "MARPA header not found; not performing MARPA ANSI-C test" )
	return()
endif()
include_directories(
	${MARPA_INCLUDE}
)
find_library( MARPA_LIBS marpa )
if( MARPA_LIBS STREQUAL "MARPA_LIBS-NOTFOUND" )
	message( WARNING "MARPA library not found; not performing MARPA ANSI-C test" )
	return()
endif()
if( MSTA_FOUND )
	add_custom_command( OUTPUT test.c COMMAND ${MSTA} ${CMAKE_CURRENT_SOURCE_DIR}/test.y )
	add_custom_target( test_c DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/test.c )
elseif( BISON_FOUND )
	bison_target( test_c test.y "${CMAKE_CURRENT_BINARY_DIR}/test.c" )
else()
	message( WARNING "None of MSTA or bison found; not performing MARPA or Lotsawa ANSI-C test" )
	return()
endif()

add_executable( ansic-lotsawa test.c ${BISON_test_c_OUTPUTS} )
target_compile_definitions( ansic-lotsawa PRIVATE LOTSAWA )
add_dependencies( ansic-lotsawa ansic_c )
target_link_libraries( ansic-lotsawa
	${LOTSAWA_LIBS}
	ticker
	yaep
      )
cmake_path( GET LOTSAWA_LIBS PARENT_PATH LOTSAWA_LIB_DIR )
target_link_directories( ansic-lotsawa PRIVATE ${LOTSAWA_LIB_DIR} )
add_test( ansic-test-lotsawa-small ${SHELL} -c "${TIMER} ${CMAKE_CURRENT_BINARY_DIR}/ansic-lotsawa < ${TEST_DATA}" )
add_test( ansic-test-lotsawa-big ${SHELL} -c "${TIMER} ${CMAKE_CURRENT_BINARY_DIR}/ansic-lotsawa < ${BIGTEST_DATA}" )

add_executable( ansic-marpa test.c ${BISON_test_c_OUTPUTS} )
add_dependencies( ansic-marpa ansic_c )
target_link_libraries( ansic-marpa
	${MARPA_LIBS}
	ticker
	yaep
)
add_test( ansic-test-marpa-small ${SHELL} -c "${TIMER} ${CMAKE_CURRENT_BINARY_DIR}/ansic-marpa < ${TEST_DATA}" )
if( DEFINED ENV{MARPA_BIG_TEST})
    add_test( ansic-test-marpa-big ${SHELL} -c "${TIMER} ${CMAKE_CURRENT_BINARY_DIR}/ansic-marpa < ${BIGTEST_DATA}" )
else()
    message("!!! To test MARPA on the biggest file, define env. variable MARPA_BIG_TEST.")
    message("!!! Please be aware -- MARPA uses 30GB for this test.")
endif()
