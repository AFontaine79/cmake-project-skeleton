# Save the library directory path for use with our `add_library` call below
set(LIBRARY_ROOT_DIR ${CMAKE_CURRENT_LIST_DIR})

# All targets in the lib tree should be included with a directory prefix:
#	#include <lib/header.h>
# To accomplish this, we automatically set `target_include_directories` for library
# targets so this scheme can be used.
#
# Library targets themselves should NOT set public/interface includes within the library tree.
function(add_library target)
	# Forward all arguments to the orginal add_library
	_add_library(${target} ${ARGN})
	message("This is called for ${target} with dir: ${CMAKE_CURRENT_LIST_DIR}")
	# Ensure this directory is included as an interface include
	target_include_directories(${target} PUBLIC ${LIBRARY_ROOT_DIR})
endfunction()

add_subdirectory(example)
