cmake_minimum_required(VERSION 3.2)

project(NuttX-tools LANGUAGES C)

add_executable(mkconfig mkconfig.c cfgdefine.c)
install(TARGETS mkconfig DESTINATION bin)

add_executable(mkversion mkversion.c cfgdefine.c)
install(TARGETS mkversion DESTINATION bin)

