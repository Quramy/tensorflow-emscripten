# Config
# one of: html, js, c
FINAL_PRODUCT := js
ALLOCATE_MEM_UPFRONT := true
SINGLE_THREAD := true

# to compile to wasm, you must be on the emscripten incoming branch
# https://github.com/kripken/emscripten/wiki/WebAssembly#setup
# Also, the version of Eigen that ships with TF v1.0.0 doesn't compile with
# Clang 4.0, you may have to perform the following manual update:
# https://bitbucket.org/eigen/eigen/diff/unsupported/Eigen/CXX11/src/Tensor/TensorStorage.h?diff2=8dd2d6552a87&at=default
WEB_ASSEMBLY := false

# verify consistency
ifneq (,$(filter $(FINAL_PRODUCT),html js))
ifneq ($(SINGLE_THREAD),true)
$(error Must be single threaded to compile to HTML or JS)
endif
endif

ifeq ($(WEB_ASSEMBLY),true)
ifeq (,$(filter $(FINAL_PRODUCT),html js))
$(error Must be HTML or JS to compile to WASM)
endif
endif

# Find where we're running from, so we can store generated files here.
MAKEFILE_DIR := $(shell dirname $(realpath $(lastword $(MAKEFILE_LIST))))
HAS_GEN_HOST_PROTOC := $(shell test -f $(MAKEFILE_DIR)/gen/protobuf-host/bin/protoc && echo "true" || echo "false")

# Host compilation settings

# Try to figure out the host system
HOST_OS :=
ifeq ($(OS),Windows_NT)
	HOST_OS = WINDOWS
else
	UNAME_S := $(shell uname -s)
	ifeq ($(UNAME_S),Linux)
	        HOST_OS := LINUX
	endif
	ifeq ($(UNAME_S),Darwin)
		HOST_OS := OSX
	endif
endif

# Where compiled objects are stored.
HOST_OBJDIR := $(MAKEFILE_DIR)/gen/host_obj/
HOST_BINDIR := $(MAKEFILE_DIR)/gen/host_bin/
HOST_GENDIR := $(MAKEFILE_DIR)/gen/host_obj/

# Settings for the host compiler.
HOST_CXX := $(CC_PREFIX) gcc
HOST_CXXFLAGS := --std=c++11
HOST_LDOPTS :=
ifeq ($(HAS_GEN_HOST_PROTOC),true)
	HOST_LDOPTS += -L$(MAKEFILE_DIR)/gen/protobuf-host/lib
endif
HOST_LDOPTS += -L/usr/local/lib

HOST_INCLUDES := \
-I. \
-I$(MAKEFILE_DIR)/downloads/ \
-I$(MAKEFILE_DIR)/downloads/eigen/ \
-I$(MAKEFILE_DIR)/downloads/gemmlowp/ \
-I$(HOST_GENDIR)
ifeq ($(HAS_GEN_HOST_PROTOC),true)
	HOST_INCLUDES += -I$(MAKEFILE_DIR)/gen/protobuf-host/include
endif
# This is at the end so any globally-installed frameworks like protobuf don't
# override local versions in the source tree.
HOST_INCLUDES += -I/usr/local/include

HOST_LIBS := \
-lstdc++ \
-lprotobuf \
-lpthread \
-lm \
-lz

# If we're on Linux, also link in the dl library.
ifeq ($(HOST_OS),LINUX)
	HOST_LIBS += -ldl -lpthread
endif

# proto_text is a tool that converts protobufs into a form we can use more
# compactly within TensorFlow. It's a bit like protoc, but is designed to
# produce a much more minimal result so we can save binary space.
# We have to build it on the host system first so that we can create files
# that are needed for the runtime building.
PROTO_TEXT := $(HOST_BINDIR)proto_text
# The list of dependencies is derived from the Bazel build file by running
# the gen_file_lists.sh script on a system with a working Bazel setup.
PROTO_TEXT_CC_FILES := $(shell cat $(MAKEFILE_DIR)/proto_text_cc_files.txt)
PROTO_TEXT_PB_CC_LIST := $(shell cat $(MAKEFILE_DIR)/proto_text_pb_cc_files.txt)
PROTO_TEXT_PB_H_LIST := $(shell cat $(MAKEFILE_DIR)/proto_text_pb_h_files.txt)

# Locations of the intermediate files proto_text generates.
PROTO_TEXT_PB_H_FILES := $(addprefix $(HOST_GENDIR), $(PROTO_TEXT_PB_H_LIST))
PROTO_TEXT_CC_OBJS := $(addprefix $(HOST_OBJDIR), $(PROTO_TEXT_CC_FILES:.cc=.o))
PROTO_TEXT_PB_OBJS := $(addprefix $(HOST_OBJDIR), $(PROTO_TEXT_PB_CC_LIST:.cc=.o))
PROTO_TEXT_OBJS := $(PROTO_TEXT_CC_OBJS) $(PROTO_TEXT_PB_OBJS)

# Target device settings.

# Default to running on the same system we're compiling on.
# You should override TARGET on the command line if you're cross-compiling, e.g.
# make -f tensorflow/contrib/makefile/Makefile TARGET=ANDROID
TARGET := $(HOST_OS)

# Where compiled objects are stored.
GENDIR := $(MAKEFILE_DIR)/gen/
OBJDIR := $(GENDIR)obj/
LIBDIR := $(GENDIR)lib/
BINDIR := $(GENDIR)bin/
PBTGENDIR := $(GENDIR)proto_text/
PROTOGENDIR := $(GENDIR)proto/
DEPDIR := $(GENDIR)dep/
$(shell mkdir -p $(DEPDIR) >/dev/null)

# Settings for the target compiler.
CXX := $(CC_PREFIX) emcc
ifeq ($(FINAL_PRODUCT),c)
  CXX := $(CC_PREFIX) gcc
endif

OPTFLAGS := -O2
CFLAGS := -m64 -DIS_SLIM_BUILD -fexceptions -DNDEBUG -DNOTFDBG $(OPTFLAGS)

ifeq ($(SINGLE_THREAD),true)
	CFLAGS += -D__SINGLE_THREAD__
endif

ifneq (,$(filter $(FINAL_PRODUCT),html js))
  CFLAGS += -s USE_ZLIB=1 -D__MAKEFILE_JS__ -s DEMANGLE_SUPPORT=1 -s ASSERTIONS=1
	ifeq ($(ALLOCATE_MEM_UPFRONT),true)
		CFLAGS += -s TOTAL_MEMORY=1073741824
	else
		CFLAGS += -s ALLOW_MEMORY_GROWTH=1
	endif
endif

ifeq ($(FINAL_PRODUCT),html)
	CFLAGS += --emrun
	CFLAGS += --preload-file $(MAKEFILE_DIR)/js_working_directory@/
	CFLAGS += -D__MAKEFILE_JS_MAKE_HTML__
endif

CXXFLAGS := $(CFLAGS) --std=c++11
LDFLAGS := \
-L/usr/local/lib
DEPFLAGS = -MT $@ -MMD -MP -m64 -MF $(DEPDIR)/$*.Td

INCLUDES := \
-I. \
-I$(MAKEFILE_DIR)/downloads/ \
-I$(MAKEFILE_DIR)/downloads/eigen/ \
-I$(MAKEFILE_DIR)/downloads/gemmlowp/ \
-I$(PROTOGENDIR) \
-I$(PBTGENDIR) \
-I./bazel-genfiles/ \
-I./bazel-tensorflow-emscripten/

ifeq ($(HAS_GEN_HOST_PROTOC),true)
	INCLUDES += -I$(MAKEFILE_DIR)/gen/protobuf-host/include
endif

# This is at the end so any globally-installed frameworks like protobuf don't
# override local versions in the source tree.
INCLUDES += -I/usr/local/include

LIBS := \
-lstdc++ \
-lm

ifeq ($(FINAL_PRODUCT),c)
LIBS += \
-lprotobuf \
-lz
endif

ifeq ($(HAS_GEN_HOST_PROTOC),true)
	PROTOC := $(MAKEFILE_DIR)/gen/protobuf-host/bin/protoc
else
	PROTOC := protoc
endif

$(info PROTOC = "$(PROTOC)")
$(info CC_PREFIX = "$(CC_PREFIX)")

PROTOCFLAGS :=
AR := emar
ARFLAGS := -r
LIBFLAGS :=

# If we're on OS X, make sure that globals aren't stripped out.
ifeq ($(TARGET),OSX)
	LDFLAGS += -all_load
endif
# Make sure that we don't strip global constructors on Linux.
ifeq ($(TARGET),LINUX)
ifeq ($(HAS_GEN_HOST_PROTOC),true)
	# LIBFLAGS += -L$(MAKEFILE_DIR)/gen/protobuf-host/lib
	# export LD_LIBRARY_PATH=$(MAKEFILE_DIR)/gen/protobuf-host/lib
endif
	# CXXFLAGS += -fPIC
	# LIBFLAGS += -Wl,--allow-multiple-definition -Wl,--whole-archive
	# LDFLAGS := -Wl,--no-whole-archive
endif
# If we're on Linux, also link in the dl library.
ifeq ($(TARGET),LINUX)
	# LIBS += -ldl -lpthread
endif

# This library is the main target for this makefile. It will contain a minimal
# runtime that can be linked in to other programs.
LIB_NAME := libtensorflow-core.a
LIB_PATH := $(LIBDIR)$(LIB_NAME)

# A small example program that shows how to link against the library.
FINAL_EXTENSION := .js
ifeq ($(FINAL_PRODUCT),html)
	FINAL_EXTENSION := .html
endif
ifeq ($(FINAL_PRODUCT),c)
	FINAL_EXTENSION := .o
endif

GR_NAME := $(BINDIR)graph_runner$(FINAL_EXTENSION)
BM_MNIST_NAME := $(BINDIR)benchmark_mnist$(FINAL_EXTENSION)
BM_INCEPTION_NAME := $(BINDIR)benchmark_inception$(FINAL_EXTENSION)

# What sources we want to compile, derived from the main Bazel build using the
# gen_file_lists.sh script.

CORE_CC_ALL_SRCS := \
$(wildcard tensorflow/core/*.cc) \
$(wildcard tensorflow/core/common_runtime/*.cc) \
$(wildcard tensorflow/core/framework/*.cc) \
$(wildcard tensorflow/core/graph/*.cc) \
$(wildcard tensorflow/core/lib/*/*.cc) \
$(wildcard tensorflow/core/platform/*.cc) \
$(wildcard tensorflow/core/platform/*/*.cc) \
$(wildcard tensorflow/core/platform/*/*/*.cc) \
$(wildcard tensorflow/core/util/*.cc) \
$(wildcard tensorflow/core/util/*/*.cc) \
tensorflow/core/util/version_info.cc

# Remove duplicates (for version_info.cc)
CORE_CC_ALL_SRCS := $(sort $(CORE_CC_ALL_SRCS))
CORE_CC_EXCLUDE_SRCS := \
$(wildcard tensorflow/core/*/*test.cc) \
$(wildcard tensorflow/core/*/*testutil*) \
$(wildcard tensorflow/core/*/*testlib*) \
$(wildcard tensorflow/core/*/*main.cc) \
$(wildcard tensorflow/core/*/*/*test.cc) \
$(wildcard tensorflow/core/*/*/*testutil*) \
$(wildcard tensorflow/core/*/*/*testlib*) \
$(wildcard tensorflow/core/*/*/*main.cc) \
$(wildcard tensorflow/core/debug/*.cc) \
$(wildcard tensorflow/core/framework/op_gen_lib.cc) \
$(wildcard tensorflow/core/graph/dot.*) \
$(wildcard tensorflow/core/lib/gif/*) \
$(wildcard tensorflow/core/lib/io/zlib*) \
$(wildcard tensorflow/core/lib/io/record*) \
$(wildcard tensorflow/core/lib/jpeg/*) \
$(wildcard tensorflow/core/lib/png/*) \
$(wildcard tensorflow/core/util/events_writer.*) \
$(wildcard tensorflow/core/util/reporter.*) \
$(wildcard tensorflow/core/platform/default/cuda_libdevice_path.*) \
$(wildcard tensorflow/core/platform/default/stream_executor.*) \
$(wildcard tensorflow/core/platform/default/test_benchmark.*) \
$(wildcard tensorflow/core/platform/cuda.h) \
$(wildcard tensorflow/core/platform/cuda_libdevice_path.*) \
$(wildcard tensorflow/core/platform/cloud/*) \
$(wildcard tensorflow/core/platform/google/*) \
$(wildcard tensorflow/core/platform/google/*/*) \
$(wildcard tensorflow/core/platform/jpeg.*) \
$(wildcard tensorflow/core/platform/png.*) \
$(wildcard tensorflow/core/platform/stream_executor.*) \
$(wildcard tensorflow/core/platform/windows/*) \
$(wildcard tensorflow/core/user_ops/*.cu.cc) \
$(wildcard tensorflow/core/common_runtime/gpu/*) \
$(wildcard tensorflow/core/common_runtime/gpu_device_factory.*) \
tensorflow/core/platform/profile_utils/android_armv7a_cpu_utils_helper.cc

# Filter out all the excluded files.
TF_CC_SRCS := $(filter-out $(CORE_CC_EXCLUDE_SRCS), $(CORE_CC_ALL_SRCS))

# Add in any extra files that don't fit the patterns easily
TF_CC_SRCS += tensorflow/core/common_runtime/gpu/gpu_tracer.cc

# Also include the op and kernel definitions.
TF_CC_SRCS += $(shell cat $(MAKEFILE_DIR)/tf_op_files.txt)
PBT_CC_SRCS := $(shell cat $(MAKEFILE_DIR)/tf_pb_text_files.txt)
PROTO_SRCS := $(shell cat $(MAKEFILE_DIR)/tf_proto_files.txt)
GR_SRCS := $(MAKEFILE_DIR)/programs/graph_runner/main.cc
BM_MNIST_SRCS := $(MAKEFILE_DIR)/programs/benchmarks/benchmark_mnist.cc
BM_INCEPTION_SRCS := $(MAKEFILE_DIR)/programs/benchmarks/benchmark_inception.cc

# File names of the intermediate files target compilation generates.
TF_CC_OBJS := $(addprefix $(OBJDIR), $(TF_CC_SRCS:.cc=.o))
PBT_GEN_FILES := $(addprefix $(PBTGENDIR), $(PBT_CC_SRCS))
PBT_OBJS := $(addprefix $(OBJDIR), $(PBT_CC_SRCS:.cc=.o))
PROTO_CC_SRCS := $(addprefix $(PROTOGENDIR), $(PROTO_SRCS:.proto=.pb.cc))
PROTO_OBJS := $(addprefix $(OBJDIR), $(PROTO_SRCS:.proto=.pb.o))
LIB_OBJS := $(PROTO_OBJS) $(TF_CC_OBJS) $(PBT_OBJS)
GR_OBJS := $(addprefix $(OBJDIR), $(GR_SRCS:.cc=.o))
BM_MNIST_OBJS := $(addprefix $(OBJDIR), $(BM_MNIST_SRCS:.cc=.o))
BM_INCEPTION_OBJS := $(addprefix $(OBJDIR), $(BM_INCEPTION_SRCS:.cc=.o))
PROTOBUF_LIB := $(MAKEFILE_DIR)/downloads/protobuf-emscripten/src/.libs/libprotobuf.a

ifeq ($(FINAL_PRODUCT),c)
	PROTOBUF_LIB :=
endif

.PHONY: clean cleantarget

# The target that's compiled if there's no command-line arguments.
all: $(LIB_PATH) $(GR_NAME) $(BM_MNIST_NAME) $(BM_INCEPTION_NAME)

# Rules for target compilation.
.phony_version_info:
tensorflow/core/util/version_info.cc: .phony_version_info
	tensorflow/tools/git/gen_git_source.sh $@

# Gathers together all the objects we've compiled into a single '.a' archive. -- Not necessary because compile data in ourselves
$(LIB_PATH): $(LIB_OBJS)
	@mkdir -p $(dir $@)
	$(AR) $(ARFLAGS) $(LIB_PATH) $(LIB_OBJS)

# for embind
BIND_FLAG := --bind
ifeq ($(FINAL_PRODUCT),c)
	BIND_FLAG :=
endif

WASM_FLAG :=
ifeq ($(WEB_ASSEMBLY), true)
	WASM_FLAG := -s WASM=1
endif

$(GR_NAME): $(GR_OBJS) $(LIB_PATH)
	@mkdir -p $(dir $@)
	$(CXX) $(CXXFLAGS) $(INCLUDES) \
	$(WASM_FLAG) \
	-o $(GR_NAME) $(GR_OBJS) \
	$(LIBFLAGS) $(LIB_OBJS) $(LDFLAGS) $(LIBS) \
	$(PROTOBUF_LIB) \
	$(BIND_FLAG)

$(BM_MNIST_NAME): $(BM_MNIST_OBJS) $(LIB_PATH)
	@mkdir -p $(dir $@)
	$(CXX) $(CXXFLAGS) $(INCLUDES) \
	$(WASM_FLAG) \
	-o $(BM_MNIST_NAME) $(BM_MNIST_OBJS) \
	$(LIBFLAGS) $(LIB_OBJS) $(LDFLAGS) $(LIBS) \
	$(PROTOBUF_LIB) \
	$(BIND_FLAG)

$(BM_INCEPTION_NAME): $(BM_INCEPTION_OBJS) $(LIB_PATH)
	@mkdir -p $(dir $@)
	$(CXX) $(CXXFLAGS) $(INCLUDES) \
	$(WASM_FLAG) \
	-o $(BM_INCEPTION_NAME) $(BM_INCEPTION_OBJS) \
	$(LIBFLAGS) $(LIB_OBJS) $(LDFLAGS) $(LIBS) \
	$(PROTOBUF_LIB) \
	$(BIND_FLAG)

# Matches on the normal hand-written TensorFlow C++ source files.
$(OBJDIR)%.o: %.cc | $(PBT_GEN_FILES)
	@mkdir -p $(dir $@)
	@mkdir -p $(dir $(DEPDIR)$*)
	$(CXX) $(CXXFLAGS) $(DEPFLAGS) $(INCLUDES) -c $< -o $@
	@mv -f $(DEPDIR)/$*.Td $(DEPDIR)/$*.d

$(OBJDIR)%.o: %.c
	@mkdir -p $(dir $@)
	@mkdir -p $(dir $(DEPDIR)$*)
	$(CXX) $(CFLAGS) $(OPTFLAGS) $(DEPFLAGS) $(INCLUDES) -c $< -o $@
	@mv -f $(DEPDIR)/$*.Td $(DEPDIR)/$*.d

# Compiles C++ source files that have been generated by protoc.
$(OBJDIR)%.pb.o: $(PROTOGENDIR)%.pb.cc
	@mkdir -p $(dir $@)
	$(CXX) $(CXXFLAGS) $(INCLUDES) -c $< -o $@

# Builds C++ code from proto files using protoc.
$(PROTOGENDIR)%.pb.cc $(PROTOGENDIR)%.pb.h: %.proto
	@mkdir -p $(dir $@)
	$(PROTOC) $(PROTOCFLAGS) $< --cpp_out $(PROTOGENDIR)

# Uses proto_text to generate minimal pb_text C++ files from protos.
$(PBTGENDIR)%.pb_text.cc $(PBTGENDIR)%.pb_text.h $(PBTGENDIR)%.pb_text-impl.h: %.proto | $(PROTO_TEXT)
	@mkdir -p $(dir $@)
	$(PROTO_TEXT) \
	$(PBTGENDIR)tensorflow/core \
	tensorflow/core/ \
	tensorflow/tools/proto_text/placeholder.txt \
	$<

# Compiles the C++ source files created by proto_text.
$(OBJDIR)%.pb_text.o: $(PBTGENDIR)%.pb_text.cc
	@mkdir -p $(dir $@)
	$(CXX) $(CXXFLAGS) $(INCLUDES) -c $< -o $@

# Makes sure that we don't compile the protoc-generated C++ sources before they
# and the proto_text files have been created.
$(PROTO_OBJS): $(PROTO_CC_SRCS) $(PBT_GEN_FILES)

# Host compilation rules.

# For normal manually-created TensorFlow C++ source files.
$(HOST_OBJDIR)%.o: %.cc
	@mkdir -p $(dir $@)
	$(HOST_CXX) $(HOST_CXXFLAGS) $(HOST_INCLUDES) -c $< -o $@

# Compiles object code from protoc-built C++ source files.
$(HOST_OBJDIR)%.pb.o: $(HOST_GENDIR)%.pb.cc
	@mkdir -p $(dir $@)
	$(HOST_CXX) $(HOST_CXXFLAGS) $(HOST_INCLUDES) -c $< -o $@

# Ensures we wait until proto_text has generated the .h files from protos before
# we compile the C++.
$(PROTO_TEXT_OBJS) : $(PROTO_TEXT_PB_H_FILES)

# Runs proto_text to generate C++ source files from protos.
$(PROTO_TEXT): $(PROTO_TEXT_OBJS) $(PROTO_TEXT_PB_H_FILES)
	@mkdir -p $(dir $@)
	$(HOST_CXX) $(HOST_CXXFLAGS) $(HOST_INCLUDES) \
	-o $(PROTO_TEXT) $(PROTO_TEXT_OBJS) $(HOST_LDOPTS) $(HOST_LIBS)

# Compiles the C++ source files from protos using protoc.
$(HOST_GENDIR)%.pb.cc $(HOST_GENDIR)%.pb.h: %.proto
	@mkdir -p $(dir $@)
	$(PROTOC) $(PROTOCFLAGS) $< --cpp_out $(HOST_GENDIR)

# Gets rid of all generated files.
clean:
	rm -rf $(MAKEFILE_DIR)/gen
	rm -rf tensorflow/core/util/version_info.cc

# Gets rid of target files only, leaving the host alone. Also leaves the lib
# directory untouched deliberately, so we can persist multiple architectures
# across builds for iOS.
cleantarget:
	rm -rf $(OBJDIR)
	rm -rf $(BINDIR)

$(DEPDIR)/%.d: ;
.PRECIOUS: $(DEPDIR)/%.d

-include $(patsubst %,$(DEPDIR)/%.d,$(basename $(TF_CC_SRCS)))
