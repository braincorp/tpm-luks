project(tpm-luks)
cmake_minimum_required (VERSION 3.5)

include(ExternalProject)

ExternalProject_Add(
  tpm-luks
  SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/
  CONFIGURE_COMMAND autoreconf -ivf COMMAND ./configure --prefix=/usr/local
  BUILD_COMMAND $(MAKE) COMMAND echo 'done'
  INSTALL_COMMAND $(MAKE) DESTDIR=${CMAKE_CURRENT_BINARY_DIR}/tpm-luks install
  BUILD_IN_SOURCE 1
)

add_custom_target(
  tpm-luks.clean
  COMMAND rm -rf ${CMAKE_CURRENT_BINARY_DIR} || $(MAKE) -C ${CMAKE_CURRENT_SOURCE_DIR} distclean
)

install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/tpm-luks/ DESTINATION / USE_SOURCE_PERMISSIONS COMPONENT boot)
