cmake_minimum_required(VERSION 3.0.0)
set(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS  TRUE)

if(COMMAND cmake_policy)
    cmake_policy(VERSION 2.6)
endif()

set(SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
set(BUILD_DIR ${CMAKE_CURRENT_BINARY_DIR})

if(NOT EXISTS ${SOURCE_DIR}/awesomeConfig.cmake)
    message(FATAL_ERROR "Please provide awesomeConfig.cmake")
endif()

# Define many variables

include(awesomeConfig.cmake)

include_directories(
    ${BUILD_DIR}
    ${AWESOME_COMMON_REQUIRED_INCLUDE_DIRS}
    ${AWESOME_REQUIRED_INCLUDE_DIRS}
    ${AWESOME_OPTIONAL_INCLUDE_DIRS})

set(CHECK_TARGETS check-integration)

set(AWE_CONF_FILE_DEFAULT ${BUILD_DIR}/awesomerc.lua)
set(AWE_CONF_FILE rc.lua)

set(AWE_ICON_DIR ${SOURCE_DIR}/icons)
set(AWE_THEMES_DIR ${SOURCE_DIR}/themes)

set(AWE_DOC_DIR ${BUILD_DIR}/docs)

set(AWE_DOC_FILES
    ${AWE_DOC_DIR}/00-authors.md
    ${AWE_DOC_DIR}/01-readme.md
    ${AWE_DOC_DIR}/02-contributing.md
    ${SOURCE_DIR}/LICENSE)

set(AWE_SRCS
    ${BUILD_DIR}/awesome.c
    ${BUILD_DIR}/banning.c
    ${BUILD_DIR}/color.c
    ${BUILD_DIR}/dbus.c
    ${BUILD_DIR}/draw.c
    ${BUILD_DIR}/event.c
    ${BUILD_DIR}/ewmh.c
    ${BUILD_DIR}/keygrabber.c
    ${BUILD_DIR}/luaa.c
    ${BUILD_DIR}/mouse.c
    ${BUILD_DIR}/mousegrabber.c
    ${BUILD_DIR}/property.c
    ${BUILD_DIR}/root.c
    ${BUILD_DIR}/selection.c
    ${BUILD_DIR}/spawn.c
    ${BUILD_DIR}/stack.c
    ${BUILD_DIR}/strut.c
    ${BUILD_DIR}/systray.c
    ${BUILD_DIR}/xwindow.c
    ${BUILD_DIR}/xkb.c
    ${BUILD_DIR}/xrdb.c
    ${BUILD_DIR}/common/atoms.c
    ${BUILD_DIR}/common/backtrace.c
    ${BUILD_DIR}/common/buffer.c
    ${BUILD_DIR}/common/luaclass.c
    ${BUILD_DIR}/common/lualib.c
    ${BUILD_DIR}/common/luaobject.c
    ${BUILD_DIR}/common/util.c
    ${BUILD_DIR}/common/version.c
    ${BUILD_DIR}/common/xcursor.c
    ${BUILD_DIR}/common/xembed.c
    ${BUILD_DIR}/common/xutil.c
    ${BUILD_DIR}/objects/button.c
    ${BUILD_DIR}/objects/client.c
    ${BUILD_DIR}/objects/drawable.c
    ${BUILD_DIR}/objects/drawin.c
    ${BUILD_DIR}/objects/key.c
    ${BUILD_DIR}/objects/screen.c
    ${BUILD_DIR}/objects/tag.c
    ${BUILD_DIR}/objects/window.c)

set(AWE_MAN_SRCS
    ${SOURCE_DIR}/manpages/awesome.1.txt
    ${SOURCE_DIR}/manpages/awesome-client.1.txt
    ${SOURCE_DIR}/manpages/awesomerc.5.txt)
set(AWE_MAN_LANGS it es fr de ru)

# Don't strip RPATH if compiling on Solaris
if (${CMAKE_SYSTEM_NAME} MATCHES "SunOS")
    set(CMAKE_SKIP_BUILD_RPATH FALSE)
    set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
    set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
endif()

add_executable(${PROJECT_AWE_NAME}
    ${AWE_SRCS})

# CFLAGS
set(AWESOME_C_FLAGS
    -O1 -std=gnu99 -ggdb3 -fno-strict-aliasing -Wall -Wextra
    -Wchar-subscripts -Wundef -Wshadow -Wcast-align -Wwrite-strings
    -Wsign-compare -Wunused -Wno-unused-parameter -Wuninitialized -Winit-self
    -Wpointer-arith -Wformat-nonliteral
    -Wno-format-zero-length -Wmissing-format-attribute -Wmissing-prototypes
    -Wstrict-prototypes
    CACHE STRING "CFLAGS used to compile ${PROJECT_AWE_NAME}")
mark_as_advanced(AWESOME_C_FLAGS)
target_compile_options(${PROJECT_AWE_NAME} PRIVATE ${AWESOME_C_FLAGS})

# Make sure awesomerc.lua is generated
add_dependencies(${PROJECT_AWE_NAME} generate_awesomerc)

# Linux w/ GCC requires -rdynamic to get backtrace to fully work.
#
# For "historical reasons", CMake adds the option to the linker flags
# unnoticeably for Linux w/ GCC through its modules Linux-GNU.cmake
# and Linux-GNU-C.cmake.  Our build system has counted on that.  But
# just in case CMake should do away with the convention suddenly...
if(DEFINED CMAKE_SHARED_LIBRARY_LINK_C_FLAGS AND
        NOT CMAKE_SHARED_LIBRARY_LINK_C_FLAGS MATCHES "-rdynamic")
    target_link_libraries(${PROJECT_AWE_NAME}
        $<$<AND:$<PLATFORM_ID:Linux>,$<C_COMPILER_ID:GNU>>:-rdynamic>)
endif()

# FreeBSD requires dynamic linking
if(${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD")
    set_target_properties(${PROJECT_AWE_NAME}
        PROPERTIES
        LINK_FLAGS -export-dynamic)
endif()

target_link_libraries(${PROJECT_AWE_NAME}
    ${AWESOME_COMMON_REQUIRED_LDFLAGS}
    ${AWESOME_REQUIRED_LDFLAGS}
    ${AWESOME_OPTIONAL_LDFLAGS})

# check for lgi and the needed gobject introspection files
add_custom_target(lgi-check ALL
    COMMAND ${SOURCE_DIR}/build-utils/lgi-check.sh)

# {{{ Generated sources
# atoms
file(MAKE_DIRECTORY ${BUILD_DIR}/common)
add_custom_command(
    COMMAND ${SOURCE_DIR}/build-utils/atoms-ext.sh ${SOURCE_DIR}/common/atoms.list
    ARGS    > ${BUILD_DIR}/common/atoms-extern.h
    OUTPUT  ${BUILD_DIR}/common/atoms-extern.h
    WORKING_DIRECTORY ${SOURCE_DIR}
    DEPENDS ${SOURCE_DIR}/common/atoms.list
    COMMENT "Generating atoms-extern.h"
    VERBATIM)

add_custom_command(
    COMMAND ${SOURCE_DIR}/build-utils/atoms-int.sh ${SOURCE_DIR}/common/atoms.list
    ARGS    > ${BUILD_DIR}/common/atoms-intern.h
    OUTPUT  ${BUILD_DIR}/common/atoms-intern.h
    WORKING_DIRECTORY ${SOURCE_DIR}
    DEPENDS ${SOURCE_DIR}/common/atoms.list
    COMMENT "Generating atoms-intern.h"
    VERBATIM)

add_custom_target(generated_sources
    DEPENDS ${BUILD_DIR}/common/atoms-intern.h
            ${BUILD_DIR}/common/atoms-extern.h)

# Default theme directory
file(MAKE_DIRECTORY ${BUILD_DIR}/themes/default)
file(MAKE_DIRECTORY ${BUILD_DIR}/themes/sky)
file(MAKE_DIRECTORY ${BUILD_DIR}/themes/zenburn)
file(MAKE_DIRECTORY ${BUILD_DIR}/themes/xresources)
add_dependencies(${PROJECT_AWE_NAME} generated_sources)
# }}}

# {{{ Version stamp
if(BUILD_FROM_GIT)
    add_custom_target(version_stamp ALL
        COMMAND ${SOURCE_DIR}/build-utils/git-version-stamp.sh
                ${VERSION_STAMP_FILE}
                ${BUILD_DIR}/awesome-version-internal.h
        WORKING_DIRECTORY ${SOURCE_DIR})

    add_dependencies(${PROJECT_AWE_NAME} version_stamp)
endif()
# }}}

# {{{ Manpages
if(GENERATE_MANPAGES)
    if(NOT BUILD_DIR STREQUAL SOURCE_DIR)
        file(MAKE_DIRECTORY ${BUILD_DIR}/manpages)
    endif()

    # add the default translation to the list of languages
    set(AWE_MAN_LANGS default ${AWE_MAN_LANGS})

    foreach(lang ${AWE_MAN_LANGS})

        foreach(txtfile ${AWE_MAN_SRCS})
            # figure the base name of the file (ie "awesome.1")
            GET_FILENAME_COMPONENT(tmpname ${txtfile} NAME)
            string(REGEX REPLACE ".txt\$" "" basename ${tmpname})

            # figure the relative path of the file
            GET_FILENAME_COMPONENT(tmppath ${txtfile} PATH)
            string(REPLACE ${SOURCE_DIR}/ "" relpath ${tmppath})

            # figure the manpage section to install to from filename
            string(REGEX REPLACE "^.*\\.([0-9])$" "\\1" section ${basename})

            # construct the language specific versions of the basename and path
            if (lang STREQUAL default)
                set(basename2 ${basename})
                set(relpath2 ${relpath}/man${section})
            else()
                set(basename2 ${basename}.${lang})
                set(relpath2 ${relpath}/${lang}/man${section})
            endif()

            # create the build directory (if it does not exist)
            file(MAKE_DIRECTORY ${BUILD_DIR}/${relpath2})

            # set the final filenames
            set(txtfile ${SOURCE_DIR}/${relpath}/${basename2}.txt)
            set(xmlfile ${BUILD_DIR}/${relpath2}/${basename}.xml)
            set(gzfile  ${BUILD_DIR}/${relpath2}/${basename}.gz)
            set(manfile ${BUILD_DIR}/${relpath2}/${basename})

            add_custom_command(
                COMMAND ${ASCIIDOC_EXECUTABLE} -d manpage -b docbook -o ${xmlfile} - < ${txtfile}
                WORKING_DIRECTORY ${BUILD_DIR}/${relpath2}
                OUTPUT  ${xmlfile}
                DEPENDS ${txtfile}
                VERBATIM)
            add_custom_command(
                COMMAND ${XMLTO_EXECUTABLE} man ${xmlfile}
                OUTPUT  ${manfile}
                WORKING_DIRECTORY ${BUILD_DIR}/${relpath2}
                DEPENDS ${xmlfile})

            if(COMPRESS_MANPAGES)
                add_custom_command(
                    COMMAND ${GZIP_EXECUTABLE} < ${manfile} > ${gzfile}
                    OUTPUT  ${gzfile}
                    WORKING_DIRECTORY ${BUILD_DIR}/${relpath2}
                    DEPENDS ${manfile}
                    VERBATIM)

                set(MAN_FILES ${MAN_FILES} ${gzfile})
            else()
                set(MAN_FILES ${MAN_FILES} ${manfile})
            endif()
        endforeach()

    endforeach()

    add_custom_target(man ALL DEPENDS ${MAN_FILES})
endif()
# }}}

# {{{ Lua API Documentation
if(GENERATE_DOC)

    if(NOT BUILD_DIR STREQUAL SOURCE_DIR)
        file(MAKE_DIRECTORY ${BUILD_DIR}/lib)
        file(MAKE_DIRECTORY ${BUILD_DIR}/doc)
    endif()

    file(GLOB_RECURSE AWE_LUA_FILES ${BUILD_DIR}/lib/*.lua)
    file(GLOB_RECURSE AWE_MD_FILES ${AWE_DOC_DIR}/*.md)
    file(GLOB_RECURSE AWE_MD_LUA_FILES RELATIVE "${SOURCE_DIR}/docs" docs/*.md.lua)
    foreach(file ${AWE_MD_LUA_FILES})
        # Remove .lua file extensions
        string(REPLACE ".lua" "" file ${file})
        list(APPEND AWE_MD_FILES "${AWE_DOC_DIR}/${file}")
    endforeach()

    # Copy the images to the build directory
    file(COPY ${SOURCE_DIR}/docs/images DESTINATION ${BUILD_DIR}/doc)

    # Copy the aliases to the build directory
    file(COPY ${SOURCE_DIR}/docs/aliases DESTINATION ${BUILD_DIR}/docs)

    add_custom_target(ldoc ALL
        COMMAND ${LDOC_EXECUTABLE} .
        WORKING_DIRECTORY ${AWE_DOC_DIR}
        DEPENDS ${AWE_LUA_FILES} ${AWE_MD_FILES}
    )

    # Run ldoc and make it fail if any warnings are generated. The
    # redirection-magic swaps stdout and stderr and awk exits with a non-zero
    # status if it sees at least one line of input.
    # All together, this fails as soon as ldoc prints on stderr.
    add_custom_target(check-ldoc-warnings
        COMMAND ${LDOC_EXECUTABLE} . 3>&1 1>&2 2>&3 | awk "{ fail=1 \; print } END { exit fail }"
        WORKING_DIRECTORY ${AWE_DOC_DIR}
        DEPENDS ${AWE_LUA_FILES} ${AWE_MD_FILES}
    )
    list(APPEND CHECK_QA_TARGETS check-ldoc-warnings)
endif()
# }}}

# {{{ Theme icons
file(GLOB icon_sources RELATIVE ${SOURCE_DIR} ${SOURCE_DIR}/themes/*/titlebar/*.png)

foreach(icon ${icon_sources})
    # Copy all icons to the build dir to simplify the following code.
    # Source paths are interpreted relative to ${SOURCE_DIR}, target paths
    # relative to ${BUILD_DIR}.
    get_filename_component(icon_path ${icon} PATH)
    get_filename_component(icon_name ${icon} NAME)
    file(COPY ${icon} DESTINATION ${icon_path})
    set(ALL_ICONS ${ALL_ICONS} "${icon_path}/${icon_name}")
endforeach()

macro(a_icon_convert match replacement input)
    string(REPLACE ${match} ${replacement} output ${input})

    if(NOT ${input} STREQUAL ${output} AND NOT ";${ALL_ICONS};" MATCHES ";${output};")
        set(ALL_ICONS ${ALL_ICONS} ${output})

        add_custom_command(
            COMMAND ${CONVERT_EXECUTABLE} ${input} -strip ${ARGN} ${output}
            OUTPUT  ${output}
            DEPENDS ${input}
            VERBATIM)
    endif()
endmacro()

foreach(icon ${ALL_ICONS})
    # Make unfocused icons translucent
    a_icon_convert("_focus" "_normal" ${icon}
        -colorspace rgb -gamma 0.6 -channel A -evaluate Multiply 0.4)
endforeach()

foreach(icon ${ALL_ICONS})
    # Make inactive icons grayscale
    a_icon_convert("_active" "_inactive" ${icon}
        -colorspace Gray)
endforeach()

add_custom_target(generated_icons ALL DEPENDS ${ALL_ICONS})
# }}}

# {{{ Dist tarball
if(BUILD_FROM_GIT)
    add_custom_target(dist
        COMMAND ${SOURCE_DIR}/build-utils/dist.sh ${VERSION}
        WORKING_DIRECTORY ${SOURCE_DIR})
endif()
# }}}

# {{{ Installation
install(TARGETS ${PROJECT_AWE_NAME} RUNTIME DESTINATION bin)
install(FILES "utils/awesome-client" DESTINATION bin PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
install(DIRECTORY ${BUILD_DIR}/lib DESTINATION ${AWESOME_DATA_PATH}
    PATTERN "*.in" EXCLUDE)
install(FILES ${AWE_CONF_FILE_DEFAULT} DESTINATION ${AWESOME_SYSCONFDIR}
        RENAME ${AWE_CONF_FILE})
if(GENERATE_MANPAGES)
    if(COMPRESS_MANPAGES)
        set(regex "\\.(xml|txt|[0-9])$")
    else()
        set(regex "\\.(xml|txt|gz)$")
    endif()
    install(DIRECTORY ${BUILD_DIR}/${relpath}/ DESTINATION ${AWESOME_MAN_PATH}
            REGEX ${regex} EXCLUDE )
endif()
install(DIRECTORY ${AWE_ICON_DIR} DESTINATION ${AWESOME_DATA_PATH})
install(DIRECTORY ${SOURCE_DIR}/themes DESTINATION ${AWESOME_DATA_PATH}
    PATTERN "*.lua" EXCLUDE)
install(DIRECTORY ${BUILD_DIR}/themes DESTINATION ${AWESOME_DATA_PATH}
    PATTERN "*.lua")
install(FILES ${AWE_DOC_FILES}  DESTINATION ${AWESOME_DOC_PATH})
install(FILES "awesome.desktop" DESTINATION ${AWESOME_XSESSION_PATH})
if(GENERATE_DOC)
    install(DIRECTORY ${BUILD_DIR}/doc DESTINATION ${AWESOME_DOC_PATH})
endif()
# }}}

# {{{ Tests
add_custom_target(check-integration
    sh -c "CMAKE_BINARY_DIR='${CMAKE_BINARY_DIR}' ${CMAKE_SOURCE_DIR}/tests/run.sh"
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    COMMENT "Running integration tests"
    USES_TERMINAL
    VERBATIM)
a_find_program(BUSTED_EXECUTABLE busted FALSE)
if(BUSTED_EXECUTABLE)
    # Keep the arguments in sync with the version below!
    add_custom_target(check-unit ALL
        ${BUSTED_EXECUTABLE} "--helper=${CMAKE_SOURCE_DIR}/spec/preload.lua"
        "--lpath=${CMAKE_BINARY_DIR}/lib/?.lua;${CMAKE_BINARY_DIR}/lib/?/init.lua;spec/?.lua"
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        COMMENT "Running unit tests"
        VERBATIM)
    list(APPEND CHECK_TARGETS check-unit)

    # Same as above, but with --coverage argument
    add_custom_target(check-unit-coverage
        "BUILD_DIRECTORY=${CMAKE_BINARY_DIR}/"
        ${BUSTED_EXECUTABLE} "--helper=${CMAKE_SOURCE_DIR}/spec/preload.lua"
        "--lpath=${CMAKE_BINARY_DIR}/lib/?.lua;${CMAKE_BINARY_DIR}/lib/?/init.lua;spec/?.lua"
        "--coverage"
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        COMMENT "Running unit tests under LuaCov"
        VERBATIM)
endif()
a_find_program(LUACHECK_EXECUTABLE luacheck FALSE)
if(LUACHECK_EXECUTABLE)
    add_custom_target(luacheck
        ${LUACHECK_EXECUTABLE} lib spec tests themes awesomerc.lua
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        COMMENT "Running Luacheck"
        VERBATIM)
    list(APPEND CHECK_QA_TARGETS luacheck)
endif()
add_custom_target(check-qa DEPENDS ${CHECK_QA_TARGETS})
add_custom_target(check DEPENDS ${CHECK_TARGETS} check-qa)
# }}}

INCLUDE(${CMAKE_SOURCE_DIR}/Packaging.cmake)

# vim: filetype=cmake:expandtab:shiftwidth=4:tabstop=8:softtabstop=4:textwidth=80:foldmethod=marker
