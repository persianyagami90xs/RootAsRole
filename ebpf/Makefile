#Makefile for capable
#Author: Eddie BILLOIR
KERNEL ?= /lib/modules/$(shell uname -r)/build
NOSTDINC_FLAGS := -nostdinc -isystem $(shell $(CC) -print-file-name=include)
ARCH=$(shell uname -m | sed 's/x86_64/x86/' | sed 's/i386/x86/')

LINUXINCLUDE += -I$(KERNEL)/arch/$(ARCH)/include/generated/uapi
LINUXINCLUDE += -I$(KERNEL)/arch/$(ARCH)/include/generated
LINUXINCLUDE += -I$(KERNEL)/arch/$(ARCH)/include
LINUXINCLUDE += -I$(KERNEL)/arch/$(ARCH)/include/uapi
LINUXINCLUDE += -I$(KERNEL)/include
LINUXINCLUDE += -I$(KERNEL)/include/uapi
LINUXINCLUDE += -include $(KERNEL)/include/linux/kconfig.h
LINUXINCLUDE += -I$(KERNEL)/include/generated/uapi
LINUXINCLUDE += -I$(KER_DIR)
LINUXINCLUDE += -include asm_goto_workaround.h # you must precisely include the file

EXTRA_CFLAGS=-Werror

LDFLAGS= -lelf

LLC ?= llc
CLANG ?= clang
CC = gcc

SRC_DIR := src
SR_SRC_DIR := ../src
OBJ_DIR := obj
SR_OBJ_DIR := ../obj
BIN_DIR := bin
SR_BIN_DIR := ../bin
KER_DIR := src/kernel

COMPOPTIONS = -O2 -Wall -pedantic -I ./ -g
LDOPTIONS := -O2 -Wall -pedantic -lcap -lcap-ng -I ./ -lelf -g 
EXECUTABLE := capable

KERN_SOURCES = $(KER_DIR)/${EXECUTABLE:=_kern.c}
KERN_OBJECTS = $(BIN_DIR)/$(EXECUTABLE:=_kern.o)
OBJS := $(addprefix $(OBJ_DIR)/,bpf.o bpf_load.o capable.o sorting.o)
SR_OBJS:= $(addprefix $(SR_OBJ_DIR)/,capabilities.o)
BINS := $(BIN_DIR)/$(EXECUTABLE)

all: $(BINS) $(KERN_OBJECTS)

.PHONY: clean

$(OBJ_DIR)/%.o: $(SRC_DIR)/%.c | $(OBJ_DIR)
	$(CC) $(COMPOPTIONS) -o $@ -c $<

$(SR_OBJ_DIR)/%.o : $(SR_SRC_DIR)/%.c | $(SR_OBJ_DIR)
	$(CC) $(COMPOPTIONS) -o $@ -c $<

$(OBJS): | $(OBJ_DIR)

$(SR_OBJS): | $(SR_OBJ_DIR)

$(OBJ_DIR):
	mkdir $(OBJ_DIR)

$(SR_OBJ_DIR):
	mkdir $(SR_OBJ_DIR)

$(BIN_DIR)/$(EXECUTABLE): $(SR_OBJS) $(OBJS) | $(SR_BIN_DIR) $(BIN_DIR)
	$(CC) $^ -o $@ $(LDOPTIONS)

$(BINS): | $(BIN_DIR)

$(BIN_DIR):
	mkdir $(BIN_DIR)

$(SR_BIN_DIR):
	mkdir $(SR_BIN_DIR)

build: $(BIN_DIR)/$(EXECUTABLE)

install: $(BINS) $(KERN_OBJECTS)
	id -u capable &>/dev/null || useradd capable -MNr -s /usr/sbin/nologin -d /nonexistent -c "Capable from RootAsRole,,,"
	mkdir -p /usr/lib/RootAsRole&&cp $(BIN_DIR)/$(EXECUTABLE:=_kern.o) /usr/lib/RootAsRole
	cp $(BIN_DIR)/$(EXECUTABLE) /usr/bin
	cp $(BIN_DIR)/capable_kern.o /usr/lib/RootAsRole
	setcap cap_dac_override,cap_sys_admin,cap_setuid+ep /usr/bin/capable

clean:
	@rm -rf $(BIN_DIR) $(OBJ_DIR) $(SR_OBJS)

$(KERN_OBJECTS): $(KER_DIR)/capable_kern.c
	#it will generate .ll file which is actually a LLVM assembly code
	$(CLANG) -S $(NOSTDINC_FLAGS) $(LINUXINCLUDE) $(EXTRA_CFLAGS) \
	    -D__KERNEL__ -D__ASM_SYSREG_H -Wno-unused-value -Wno-pointer-sign \
	    -Wno-compare-distinct-pointer-types \
	    -Wno-gnu-variable-sized-type-not-at-end \
	    -Wno-tautological-compare \
	    -O2 -emit-llvm -c $< -o $(OBJ_DIR)/capable_kern.ll
	#now translate LLVM assembly to native assembly
	$(LLC) -march=bpf -filetype=obj -o $(BIN_DIR)/$(EXECUTABLE:=_kern.o) $(OBJ_DIR)/capable_kern.ll