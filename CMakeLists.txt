cmake_minimum_required(VERSION 3.21)
project(stub_library VERSION 1.0.0 LANGUAGES Fortran)
set(CMAKE_SHARED_LIBRARY_PREFIX "")
add_library(for_mac_ftn_alloc SHARED stub.F90)
