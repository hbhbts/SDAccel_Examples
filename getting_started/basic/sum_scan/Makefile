# sum_scan Application
COMMON_REPO:=../../../
include $(COMMON_REPO)/utility/boards.mk
include $(COMMON_REPO)/libs/xcl/xcl.mk
include $(COMMON_REPO)/libs/opencl/opencl.mk

# sum_scan Host Application
sum_scan_SRCS=./src/sum_scan.cpp $(xcl_SRCS)
sum_scan_HDRS=$(xcl_HDRS)
sum_scan_CXXFLAGS=-I./src/ $(xcl_CXXFLAGS) $(opencl_CXXFLAGS)
sum_scan_LDFLAGS=$(opencl_LDFLAGS)

EXES=sum_scan

# sum_scan Kernel
krnl_sum_scan_SRCS=./src/krnl_sum_scan.cl
krnl_sum_scan_NDEVICES=xilinx:xil-accel-rd-ku115:4ddr-xpr:3.3 xilinx:adm-pcie-ku3:2ddr-xpr:3.3

XOS=krnl_sum_scan

# sum_scan xclbin
krnl_sum_scan_XOS=krnl_sum_scan
krnl_sum_scan_NDEVICES=xilinx:xil-accel-rd-ku115:4ddr-xpr:3.3 xilinx:adm-pcie-ku3:2ddr-xpr:3.3

XCLBINS=krnl_sum_scan

# check
check_EXE=sum_scan
check_XCLBINS=krnl_sum_scan
check_NDEVICES=$(krnl_sum_scan_NDEVICES)

CHECKS=check

#Changing Default Device for this example
DEVICES:=xilinx:adm-pcie-7v3:1ddr:3.0

include $(COMMON_REPO)/utility/rules.mk

