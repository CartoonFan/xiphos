# Xiphos build script
#
# Copyright (C) 2018 Xiphos Development Team
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 2 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU Library General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA 02110-1301, USA.
#

# building xiphos

message (STATUS "Configuring xiphos target")

# generate locale_set.c:

# convert po/LINGUAS file into a C source file containing full locale names.
# names in LINGUAS are either 5-char full "language_LOCALE" names or
# 2-char "language-only" names.  convert the latter.

# This section is, for portability reasons, a rewrite in CMake code
# of src/gtk/make-locale-set.sh.

# 1. read po/LINGUAS
file (READ ${PROJECT_SOURCE_DIR}/po/LINGUAS linguas)
# transform list of languages in a regular CMake list
string (REGEX MATCHALL "[a-zA-Z_]+" langs "${linguas}")

# 2. create header of locate_set.c
set (locale_set ${CMAKE_CURRENT_BINARY_DIR}/locale_set.c)
file (WRITE ${locale_set} "
/* File generated by CMake - Do not edit! */
char *locale_set[] = {\n"
  )
# 3. insert locale names
foreach(LANG ${langs})
  # special case, english.
  if (LANG STREQUAL "en_GB")
    file (APPEND ${locale_set} "  \"en_GB\",\n")
    file (APPEND ${locale_set} "  \"en_US\",\n")
  elseif (LANG STREQUAL "cs")
    # special case, czech.
    file (APPEND ${locale_set} "  \"cs_CZ\",\n")
  elseif (LANG STREQUAL "fa")
    # special case, iran.
    file (APPEND ${locale_set} "  \"fa_IR\",\n")
  elseif(LANG STREQUAL "he")
    # special case, israel.
    file (APPEND ${locale_set} "  \"he_IL\",\n")
  elseif(LANG STREQUAL "nb")
    # special case, norway.
    file (APPEND ${locale_set} "  \"nb_NO\",\n")
  elseif(LANG STREQUAL "sl")
    # special case, slovenia.
    file (APPEND ${locale_set} "  \"sl_SI\",\n")
  elseif(LANG STREQUAL "sv")
    # special case, sweden.
    file (APPEND ${locale_set} "  \"sv_SE\",\n")
  else ()
    string (LENGTH ${LANG} LANG_LEN)
    if (LANG_LEN LESS 5)
      # special case, 2-char "language-only" names.
      string (TOUPPER ${LANG} UPPERLANG)
      file (APPEND ${locale_set} "  \"${LANG}_${UPPERLANG}\",\n")
    else ()
      # common 5-char full "language_LOCALE" names
      file (APPEND ${locale_set} "  \"${LANG}\",\n")
    endif ()
  endif ()
endforeach ()
# 4. add footer to locale_set.c
file (APPEND ${locale_set} "  (char*)0
};\n"
  )
# 5. locale_set.c is tagged as generated during the build
set_source_files_properties(${CMAKE_CURRENT_BINARY_DIR}/locale_set.c PROPERTIES GENERATED 1)


# build xiphos
add_executable (xiphos
  about_modules.c
  about_sword.c
  about_trans.c
  about_xiphos.c
  bibletext.c
  bibletext_dialog.c
  bookmark_dialog.c
  bookmarks_menu.c
  bookmarks_treeview.c
  cipher_key_dialog.c
  commentary.c
  commentary_dialog.c
  dialog.c
  dictlex.c
  dictlex_dialog.c
  display_info.c
  dummy.cpp
  export_bookmarks.c
  export_dialog.c
  find_dialog.c
  font_dialog.c
  gbs.c
  gbs_dialog.c
  gui.c
  main_menu.c
  main_window.c
  menu_popup.c
  mod_mgr.c
  navbar_book.c
  navbar_book_dialog.c
  navbar_versekey.c
  navbar_versekey_dialog.c
  navbar_versekey_editor.c
  navbar_versekey_parallel.c
  parallel_dialog.c
  parallel_tab.c
  parallel_view.c
  preferences_dialog.c
  search_dialog.c
  search_sidebar.c
  sidebar.c
  sidebar_dialog.c
  splash.c
  tabbed_browser.c
  treekey-editor.c
  utilities.c
  xiphos.c
  ${CMAKE_CURRENT_BINARY_DIR}/locale_set.c
  )

if (DBUS)
  # need to generate 3 files: marshal.c marshal.h ipc-interface.h
  add_custom_command (
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/marshal.h
    ${CMAKE_CURRENT_BINARY_DIR}/marshal.c
    ${CMAKE_CURRENT_BINARY_DIR}/ipc-interface.h

    # process marshal.list to generate marshal.h
    COMMAND ${GLIB_GENMARSHAL}
    ${CMAKE_CURRENT_SOURCE_DIR}/marshal.list
    --prefix=ipc_marshal
    --header > ${CMAKE_CURRENT_BINARY_DIR}/marshal.h
    MAIN_DEPENDENCY ${CMAKE_CURRENT_SOURCE_DIR}/marshal.list

    # process marshal.list to generate marschal.c
    COMMAND ${GLIB_GENMARSHAL}
    ${CMAKE_CURRENT_SOURCE_DIR}/marshal.list
    --prefix=ipc_marshal
    --body > ${CMAKE_CURRENT_BINARY_DIR}/marshal.c
    MAIN_DEPENDENCY ${CMAKE_CURRENT_SOURCE_DIR}/marshal.list

    # process ipc-interface.xml to generate ipc-interface.h
    COMMAND ${DBUS_BINDING_TOOL}
    --mode=glib-server
    --output=${CMAKE_CURRENT_BINARY_DIR}/ipc-interface.h
    --prefix=ipc_object ${CMAKE_CURRENT_SOURCE_DIR}/ipc-interface.xml
    MAIN_DEPENDENCY ${CMAKE_CURRENT_SOURCE_DIR}/ipc-interface.xml

    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    COMMENT "Generating src/gtk files"
    )

  # add files to the sources
  target_sources (xiphos
    PRIVATE
    ipc.c
    ${CMAKE_CURRENT_BINARY_DIR}/marshal.c
    )
  target_include_directories (xiphos
    PRIVATE ${CMAKE_CURRENT_BINARY_DIR}
    PkgConfig::DBus
    )
  target_link_libraries(xiphos
    PRIVATE
    PkgConfig::DBus
    )
endif (DBUS)


# specify include directories to use when compiling
target_include_directories(xiphos
  PRIVATE
  Threads::Threads
  PkgConfig::Core
  PkgConfig::Gtk
  PkgConfig::Gnome
  PkgConfig::Soup
  PkgConfig::Sword
  PkgConfig::Biblesync
  )

# specify libraries or flags to use when linking
target_link_libraries(xiphos
  PRIVATE
  Threads::Threads
  PkgConfig::Core
  PkgConfig::Gtk
  PkgConfig::Gnome
  PkgConfig::Soup
  PkgConfig::Sword
  PkgConfig::Biblesync
  Xiphos::Main
  Xiphos::Backend
  Xiphos::Editor
  Xiphos::Webkit
  Xiphos::Xiphos_html
  )

if(WK_FOUND)
  target_compile_definitions(xiphos
    PRIVATE
    PkgConfig::WK
    )
  target_link_libraries(xiphos
    PRIVATE
    PkgConfig::WK
    )
endif()

if(GTK2 AND Unix-print_FOUND)
  # add gtk+-unix-print-2.0
  target_include_directories(xiphos
    PRIVATE
    PkgConfig::Unix-print
    )
  target_link_libraries(xiphos
    PRIVATE
    PkgConfig::Unix-print
    )
endif()

if (WIN32)
  # add X11/keysymdefs.h
  target_include_directories(xiphos
    PRIVATE
    ${PROJECT_SOURCE_DIR}/win32/include
    )
  # add rc file
  target_sources(xiphos
    PRIVATE
    xiphos-winres.rc
    )
endif ()


# install executable
install (TARGETS ${PROJECT_NAME}
  DESTINATION ${CMAKE_INSTALL_BINDIR}
  COMPONENT binaries)
