COMMON_REPO := ../../../

include $(COMMON_REPO)/utility/boards.mk
include $(COMMON_REPO)/libs/xcl/xcl.mk
include $(COMMON_REPO)/libs/opencl/opencl.mk

# Host Application
host_SRCS=./src/host.cpp $(xcl_SRCS)
host_HDRS=$(xcl_HDRS)
host_CXXFLAGS=-I./src/ $(xcl_CXXFLAGS) $(opencl_CXXFLAGS)
host_LDFLAGS=$(opencl_LDFLAGS)

EXES=host

# Kernel
vadd_SRCS=./src/vadd.cl
#Disabled autoPipelining for kernel as pipeline is not feasible with printf statement in kernel.
#this will allow compiler to generate design quickly as it will not attempt to pipeline.
vadd_CLFLAGS=--xp "param:compiler.enableAutoPipelining=false" -I./src/

XOS=vadd

# xclbin
vadd_XOS=vadd

XCLBINS=vadd

# check
check_EXE=host
check_XCLBINS=vadd

CHECKS=check

include $(COMMON_REPO)/utility/rules.mk

