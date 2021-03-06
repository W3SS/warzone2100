project(translation)

if (CMAKE_HOST_SYSTEM_NAME MATCHES "Windows")
	# Because of a Windows-related bugfix in FindGettext.cmake in CMake 3.10,
	# CMake 3.10+ is required if compiling *on* Windows.
	cmake_minimum_required(VERSION 3.10)
endif()

find_package(Gettext 0.18)
if(NOT GETTEXT_FOUND)
	message( WARNING "Gettext is not found. Skipping language file generation. (Only the default language, English, will be available.)" )
	return()
endif()

find_program(XGETTEXT_CMD xgettext)

if(NOT XGETTEXT_CMD)
	MESSAGE( WARNING "xgettext not found. Unable to generate Language translations without Gettext." )
	return()
endif()

# GETTEXT_CREATE_TRANSLATIONS_WZ(potFile [ALL]
#								 [INSTALL_DESTINATION <destdir>]
#								 [INSTALL_COMPONENT component]
#								 [MSGMERGE_OPTIONS <mergeoptions>]
#								 POFILES file1 ...  fileN)
#
# Starting point: https://github.com/Kitware/CMake/blob/fb3a608f1ab4a8fda04d5c2fb5ad99e396ebd1e6/Modules/FindGettext.cmake
# Distributed under the OSI-approved BSD 3-Clause License. See https://cmake.org/licensing for details.
#
# Customized for WZ to:
#	- Specify options to msgmerge
#	- Specify an install destination
#	- Specify an install component
#	- Specify a list of po files as POFILES
#	- Specify a distinct output file location for updated .po files in the build directory
#	  (instead of updating in-place, which modifies the source folder)
#	- Use the updated .po files in the build directory for the msgfmt command
#
function(GETTEXT_CREATE_TRANSLATIONS_WZ _potFile)
   set(_options ALL)
   set(_oneValueArgs INSTALL_DESTINATION INSTALL_COMPONENT)
   set(_multiValueArgs MSGMERGE_OPTIONS POFILES)

   CMAKE_PARSE_ARGUMENTS(_parsedArguments "${_options}" "${_oneValueArgs}" "${_multiValueArgs}" ${ARGN})

   set(_gmoFiles)
   get_filename_component(_potName ${_potFile} NAME)
   string(REGEX REPLACE "^(.+)(\\.[^.]+)$" "\\1" _potBasename ${_potName})
   get_filename_component(_absPotFile ${_potFile} ABSOLUTE)
   set(_msgmerge_options)
   if (DEFINED _parsedArguments_MSGMERGE_OPTIONS)
      set(_msgmerge_options ${_parsedArguments_MSGMERGE_OPTIONS})
   endif()

   foreach (_currentPoFile ${_parsedArguments_POFILES})
      get_filename_component(_currentPoFileName ${_currentPoFile} NAME)
      get_filename_component(_absFile ${_currentPoFile} ABSOLUTE)
      get_filename_component(_abs_PATH ${_absFile} PATH)
      get_filename_component(_lang ${_absFile} NAME_WE)
      set(_updatedPoFile ${CMAKE_CURRENT_BINARY_DIR}/${_currentPoFileName})
      set(_gmoFile ${CMAKE_CURRENT_BINARY_DIR}/${_lang}.gmo)

      add_custom_command(
         OUTPUT ${_updatedPoFile}
         COMMAND ${GETTEXT_MSGMERGE_EXECUTABLE} ${_msgmerge_options} --output-file="${_updatedPoFile}" ${_absFile} ${_absPotFile}
         DEPENDS ${_absPotFile} ${_absFile}
      )

      add_custom_command(
         OUTPUT ${_gmoFile}
         COMMAND ${GETTEXT_MSGFMT_EXECUTABLE} -o ${_gmoFile} ${_updatedPoFile}
         DEPENDS ${_absPotFile} ${_updatedPoFile}
      )

      if(_parsedArguments_INSTALL_DESTINATION)
	     if (DEFINED _parsedArguments_INSTALL_COMPONENT)
            install(FILES ${_gmoFile} COMPONENT ${_parsedArguments_INSTALL_COMPONENT} DESTINATION ${_parsedArguments_INSTALL_DESTINATION}/${_lang}/LC_MESSAGES RENAME ${_potBasename}.mo)
         else()
            install(FILES ${_gmoFile} DESTINATION ${_parsedArguments_INSTALL_DESTINATION}/${_lang}/LC_MESSAGES RENAME ${_potBasename}.mo)
         endif()
	  endif()
      list(APPEND _gmoFiles ${_gmoFile})

   endforeach ()

   if(NOT TARGET translations)
      add_custom_target(translations)
   endif()

	_GETTEXT_GET_UNIQUE_TARGET_NAME(translations uniqueTargetName)

   if(_parsedArguments_ALL)
      add_custom_target(${uniqueTargetName} ALL DEPENDS ${_gmoFiles})
   else()
      add_custom_target(${uniqueTargetName} DEPENDS ${_gmoFiles})
   endif()

   add_dependencies(translations ${uniqueTargetName})

endfunction()

# Must be set to warzone2100.pot, as the filename (without its extension) is used later
# to form the install .gmo file name (warzone2100.gmo) in each locale directory
set(_potFile ${CMAKE_CURRENT_BINARY_DIR}/warzone2100.pot)

# Configure the xgettext options
# NOTE: This matches what Makevars "XGETTEXT_OPTIONS" (and other values) provides
set(_xgettext_option_list --language=C++
	--from-code=UTF-8
	--no-wrap
	--width=1
	--keyword=_
	--keyword=N_
	--keyword=P_:1c,2
	--keyword=NP_:1c,2
	--package-name=Warzone
	--package-version=${PROJECT_VERSION}
	--msgid-bugs-address=warzone2100-project@lists.sourceforge.net)

# To preserve the same order that the makefile builds use, read in POTFILES.in
# and build a list of source paths in the same order.
file(STRINGS ${CMAKE_CURRENT_SOURCE_DIR}/POTFILES.in potfiles
	REGEX "^[^#].*"
)
foreach(potfile ${potfiles})
	list(APPEND source_translatable
		"${potfile}")
	list(APPEND source_translatable_depends
		"../${potfile}")
endforeach()

# Build the .pot file
ADD_CUSTOM_COMMAND(OUTPUT ${_potFile}
	COMMAND ${XGETTEXT_CMD} ${_xgettext_option_list} -o ${_potFile} ${source_translatable}
	DEPENDS ${source_translatable_depends}
	WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
	COMMENT "Extract translatable messages to ${_potFile}")

ADD_CUSTOM_TARGET(pot_file ALL
	DEPENDS ${_potFile})

# Build the *.gmo files from the *.po files (using the .pot file)
file (GLOB POFILES *.po)
GETTEXT_CREATE_TRANSLATIONS_WZ (${_potFile} ALL
	INSTALL_DESTINATION locale
	INSTALL_COMPONENT Languages
	MSGMERGE_OPTIONS --quiet --no-wrap --width=1
	POFILES ${POFILES})
