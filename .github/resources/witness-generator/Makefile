.PHONY: linux macos windows linux-lib macos-lib windows-lib android-lib ios-lib build clean

# ---- Arguments ----
ifndef PROJECT
$(error PROJECT is not set. Usage: make PROJECT=<name>. E.g.: make PROJECT=pol linux)
endif

# ---- Common ----
CXX             := g++
LD              := ld
PRIORITY_FLAGS  ?=
CXXFLAGS_COMMON := -std=c++11 -O3 -I. -Wno-address-of-packed-member -Dmain=circom_main
COMMON_SRCS     := main.cpp calcwit.cpp fr.cpp $(PROJECT).cpp
COMMON_OBJS     := $(COMMON_SRCS:.cpp=.o)
LIB_ONLY_SRCS   := $(PROJECT)/ffi.cpp circom_adapter.cpp
LIB_ONLY_OBJS   := $(LIB_ONLY_SRCS:.cpp=.o)
LIB_SRCS        := $(COMMON_SRCS) $(LIB_ONLY_SRCS)
LIB_OBJS        := $(COMMON_OBJS) $(LIB_ONLY_OBJS)
DEPS_HPP        := circom.hpp calcwit.hpp fr.hpp types.hpp $(PROJECT)/ffi.hpp
BIN             := $(PROJECT)
ifeq ($(OS),windows)
LIB_EXT         := .lib
LIB_PREFIX      :=
else
LIB_EXT         := .a
LIB_PREFIX      := lib
endif
LIB             := $(LIB_PREFIX)$(PROJECT)$(LIB_EXT)

# ---- Linux (x86_64 and aarch64) ----
# NOTE: linux (bin) links system gmp via -lgmp. CI uses linux-lib only, which
# relies on PRIORITY_FLAGS to prefer the gmp built from source over the system one.
linux: CXXFLAGS=$(PRIORITY_FLAGS) $(CXXFLAGS_COMMON)
linux: LDFLAGS=-static
linux: LDLIBS=-lgmp
linux: $(BIN)

linux-lib: CXXFLAGS=$(PRIORITY_FLAGS) $(CXXFLAGS_COMMON) -fPIC
linux-lib: $(LIB)

# ---- macOS ----
# NOTE: macos (bin) hardcodes Homebrew gmp for both includes and linking. CI uses
# macos-lib only, which relies on PRIORITY_FLAGS to prefer the gmp built from source.
macos: CXXFLAGS=$(PRIORITY_FLAGS) $(CXXFLAGS_COMMON) -I/opt/homebrew/include -include gmp_patch.hpp -mmacosx-version-min=11.0
macos: LDFLAGS=-Wl,-search_paths_first -Wl,-dead_strip
macos: LDLIBS=/opt/homebrew/lib/libgmp.a
macos: $(BIN)

macos-lib: CXXFLAGS=$(PRIORITY_FLAGS) $(CXXFLAGS_COMMON) -fPIC -I/opt/homebrew/include -include gmp_patch.hpp -mmacosx-version-min=11.0
macos-lib: $(LIB)

# ---- Windows (MinGW) ----
# NOTE: windows (bin) links system gmp via -lgmp. CI uses windows-lib only, which
# relies on PRIORITY_FLAGS to prefer the gmp built from source over the system one.
windows: CXXFLAGS=$(PRIORITY_FLAGS) $(CXXFLAGS_COMMON) -I/include -Duint="unsigned int"
windows: LDFLAGS=-static
windows: LDLIBS=-L/lib -lgmp -lmman
windows: $(BIN)

windows-lib: CXXFLAGS=$(PRIORITY_FLAGS) $(CXXFLAGS_COMMON) -fPIC -I/include -Duint="unsigned int"
windows-lib: $(LIB)

# ---- Android (aarch64; NDK clang++) ----
# ANDROID_NDK must point at the NDK root (GitHub runners export
# ANDROID_NDK_ROOT). ANDROID_API overrides the platform level.
ANDROID_NDK ?= $(or $(ANDROID_NDK_HOME),$(ANDROID_NDK_ROOT))
ANDROID_API ?= 26
ANDROID_BIN := $(ANDROID_NDK)/toolchains/llvm/prebuilt/linux-x86_64/bin
android-lib: CXX := $(ANDROID_BIN)/aarch64-linux-android$(ANDROID_API)-clang++
android-lib: LD := $(ANDROID_BIN)/ld.lld
android-lib: OBJCOPY := $(ANDROID_BIN)/llvm-objcopy
android-lib: CXXFLAGS=$(PRIORITY_FLAGS) $(CXXFLAGS_COMMON) -fPIC
android-lib: $(LIB)

# ---- iOS (arm64 device; Xcode toolchain) ----
# Same LP64 uint64_t/mp_limb_t mismatch as macOS — compile with gmp_patch.hpp
# force-included (the action copies it into the circuit dir).
IOS_SDK := $(shell xcrun --sdk iphoneos --show-sdk-path)
ios-lib: CXX := xcrun -sdk iphoneos clang++
ios-lib: LD := xcrun -sdk iphoneos ld
ios-lib: OBJCOPY := llvm-objcopy
ios-lib: SYM_PREFIX := _
ios-lib: CXXFLAGS=$(PRIORITY_FLAGS) $(CXXFLAGS_COMMON) -fPIC -arch arm64 -miphoneos-version-min=13.0 -isysroot $(IOS_SDK) -include gmp_patch.hpp
ios-lib: $(LIB)

# Localizes witness-generator implementation symbols to prevent conflicts when multiple circuit libraries are linked
# into the same binary.
# See CONTRIBUTING.md § "Symbol Isolation".
PUBLIC_SYMS := $(PROJECT)_generate_witness $(PROJECT)_generate_witness_from_files
LOCAL_OBJ   := $(PROJECT)_local.o
OBJCOPY     := $(if $(filter windows,$(OS)),objcopy,llvm-objcopy)
SYM_PREFIX  := $(if $(filter macos,$(OS)),_,)

# ---- Rules ----
$(BIN): $(COMMON_OBJS)
	$(CXX) $(LDFLAGS) $^ $(LDLIBS) -o $@

$(LIB): $(LIB_OBJS)
	$(LD) -r -o $(LOCAL_OBJ) $^
	$(OBJCOPY) $(foreach sym,$(PUBLIC_SYMS),--keep-global-symbol=$(SYM_PREFIX)$(sym)) $(LOCAL_OBJ)
	rm -f $@
	ar rcs $@ $(LOCAL_OBJ)
	rm $(LOCAL_OBJ)

%.o: %.cpp $(DEPS_HPP)
	$(CXX) $(CXXFLAGS) -c $< -o $@

clean:
	rm -f $(COMMON_OBJS) $(LIB_ONLY_OBJS) $(BIN) $(LIB) $(LOCAL_OBJ)
