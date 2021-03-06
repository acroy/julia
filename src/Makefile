JULIAHOME = $(abspath ..)
include $(JULIAHOME)/deps/Versions.make
include $(JULIAHOME)/Make.inc

override CFLAGS += $(JCFLAGS)
override CXXFLAGS += $(JCXXFLAGS)
override CPPFLAGS += $(JCPPFLAGS)

BUILDDIR ?= .

SRCS = \
	jltypes gf ast builtins module codegen interpreter \
	alloc dlload sys init task array dump toplevel jl_uv jlapi profile llvm-simdloop

HEADERS = julia.h julia_internal.h options.h $(wildcard support/*.h) $(LIBUV_INC)/uv.h

FLAGS = \
	-D_GNU_SOURCE -Iflisp -Isupport \
	-I$(call exec,$(LLVM_CONFIG) --includedir) \
	-I$(LIBUV_INC) -I$(build_includedir) -DLIBRARY_EXPORTS \
	-I$(BUILDDIR)
ifneq ($(USEMSVC), 1)
FLAGS += -Wall -Wno-strict-aliasing -fno-omit-frame-pointer -fvisibility=hidden -fno-common
endif

# In LLVM < 3.4, --ldflags includes both options and libraries, so use it both before and after --libs
# In LLVM >= 3.4, --ldflags has only options, and --system-libs has the libraries.
LLVMLINK = $(call exec,$(LLVM_CONFIG) --ldflags) $(call exec,$(LLVM_CONFIG) --libs) $(call exec,$(LLVM_CONFIG) --ldflags) $(call exec,$(LLVM_CONFIG) --system-libs 2> /dev/null)
ifeq ($(USE_LLVM_SHLIB),1)
LLVMLINK = $(call exec,$(LLVM_CONFIG) --ldflags) -lLLVM-$(call exec,$(LLVM_CONFIG) --version)
endif

COMMON_LIBS = -L$(build_shlibdir) -L$(build_libdir) $(LIBUV) $(LIBMOJIBAKE) $(NO_WHOLE_ARCHIVE) $(LLVMLINK) $(OSLIBS)
DEBUG_LIBS = $(WHOLE_ARCHIVE) $(BUILDDIR)/flisp/libflisp-debug.a $(WHOLE_ARCHIVE) $(BUILDDIR)/support/libsupport-debug.a $(COMMON_LIBS)
RELEASE_LIBS = $(WHOLE_ARCHIVE) $(BUILDDIR)/flisp/libflisp.a $(WHOLE_ARCHIVE) $(BUILDDIR)/support/libsupport.a $(COMMON_LIBS)

OBJS = $(SRCS:%=$(BUILDDIR)/%.o)
DOBJS = $(SRCS:%=$(BUILDDIR)/%.dbg.obj)
DEBUGFLAGS += $(FLAGS)
SHIPFLAGS += $(FLAGS)

ifeq ($(JULIAGC),MARKSWEEP)
SRCS += gc
endif

ifeq ($(USE_COPY_STACKS),1)
JCFLAGS += -DCOPY_STACKS
endif

default: release

release debug: %: libjulia-%

$(BUILDDIR):
	mkdir $(BUILDDIR)

$(BUILDDIR)/%.o: %.c $(HEADERS) | $(BUILDDIR)
	@$(call PRINT_CC, $(CC) $(CPPFLAGS) $(CFLAGS) $(SHIPFLAGS) -DNDEBUG -c $< -o $@)
$(BUILDDIR)/%.dbg.obj: %.c $(HEADERS) | $(BUILDDIR)
	@$(call PRINT_CC, $(CC) $(CPPFLAGS) $(CFLAGS) $(DEBUGFLAGS) -c $< -o $@)
$(BUILDDIR)/%.o: %.cpp $(HEADERS) | $(BUILDDIR)
	@$(call PRINT_CC, $(CXX) $(call exec,$(LLVM_CONFIG) --cxxflags) $(CPPFLAGS) $(CXXFLAGS) $(SHIPFLAGS) -c $< -o $@)
$(BUILDDIR)/%.dbg.obj: %.cpp $(HEADERS) | $(BUILDDIR)
	@$(call PRINT_CC, $(CXX) $(call exec,$(LLVM_CONFIG) --cxxflags) $(CPPFLAGS) $(CXXFLAGS) $(DEBUGFLAGS) -c $< -o $@)

$(BUILDDIR)/julia_flisp.boot.inc: $(BUILDDIR)/julia_flisp.boot $(BUILDDIR)/flisp/libflisp.a
	@$(call PRINT_FLISP, $(call spawn,$(BUILDDIR)/flisp/flisp) ./bin2hex.scm < $< > $@)

export julia_flisp.boot=$(BUILDDIR)/julia_flisp.boot
$(julia_flisp.boot): julia-parser.scm julia-syntax.scm \
	match.scm utils.scm jlfrontend.scm mk_julia_flisp_boot.scm $(BUILDDIR)/flisp/libflisp.a
	@$(call PRINT_FLISP, $(call spawn,$(BUILDDIR)/flisp/flisp) ./mk_julia_flisp_boot.scm)

$(BUILDDIR)/ast.o $(BUILDDIR)/ast.dbg.obj: $(BUILDDIR)/julia_flisp.boot.inc flisp/*.h
$(BUILDDIR)/codegen.o $(BUILDDIR)/codegen.dbg.obj: intrinsics.cpp debuginfo.cpp cgutils.cpp ccall.cpp disasm.cpp
$(BUILDDIR)/builtins.o $(BUILDDIR)/builtins.dbg.obj: table.c

$(BUILDDIR)/support/libsupport.a: support/*.h support/*.c
	$(MAKE) -C support BUILDDIR='$(abspath $(BUILDDIR)/support)'

$(BUILDDIR)/support/libsupport-debug.a: support/*.h support/*.c
	$(MAKE) -C support debug BUILDDIR='$(abspath $(BUILDDIR)/support)'

$(BUILDDIR)/flisp/libflisp.a: flisp/*.h flisp/*.c $(BUILDDIR)/support/libsupport.a
	$(MAKE) -C flisp BUILDDIR='$(abspath $(BUILDDIR)/flisp)'

$(BUILDDIR)/flisp/libflisp-debug.a: flisp/*.h flisp/*.c $(BUILDDIR)/support/libsupport-debug.a
	$(MAKE) -C flisp debug BUILDDIR='$(abspath $(BUILDDIR)/flisp)'

ifneq ($(USEMSVC), 1)
CXXLD = $(CXX) -shared
else
CXXLD = $(LD) -dll -export:jl_setjmp -export:jl_longjmp
endif

$(build_shlibdir)/libjulia-debug.$(SHLIB_EXT): julia.expmap $(DOBJS) $(BUILDDIR)/flisp/libflisp-debug.a $(BUILDDIR)/support/libsupport-debug.a $(LIBUV)
	@$(call PRINT_LINK, $(CXXLD) $(CXXLDFLAGS) $(DEBUGFLAGS) $(DOBJS) $(RPATH_ORIGIN) -o $@ $(LDFLAGS) $(JLIBLDFLAGS) $(DEBUG_LIBS))
	$(INSTALL_NAME_CMD)libjulia-debug.$(SHLIB_EXT) $@
	$(DSYMUTIL) $@
$(BUILDDIR)/libjulia-debug.a: julia.expmap $(DOBJS) $(BUILDDIR)/flisp/libflisp-debug.a $(BUILDDIR)/support/libsupport-debug.a
	rm -f $@
	@$(call PRINT_LINK, ar -rcs $@ $(DOBJS))
libjulia-debug: $(build_shlibdir)/libjulia-debug.$(SHLIB_EXT)

ifeq ($(SHLIB_EXT), so)
  SONAME = -Wl,-soname=libjulia.so
else
  SONAME =
endif

$(build_shlibdir)/libjulia.$(SHLIB_EXT): julia.expmap $(OBJS) $(BUILDDIR)/flisp/libflisp.a $(BUILDDIR)/support/libsupport.a $(LIBUV)
	@$(call PRINT_LINK, $(CXXLD) $(CXXLDFLAGS) $(SHIPFLAGS) $(OBJS) $(RPATH_ORIGIN) -o $@ $(LDFLAGS) $(JLIBLDFLAGS) $(RELEASE_LIBS) $(SONAME)) $(CXXLDFLAGS)
	$(INSTALL_NAME_CMD)libjulia.$(SHLIB_EXT) $@
	$(DSYMUTIL) $@
$(BUILDDIR)/libjulia.a: julia.expmap $(OBJS) $(BUILDDIR)/flisp/libflisp.a $(BUILDDIR)/support/libsupport.a
	rm -f $@
	@$(call PRINT_LINK, ar -rcs $@ $(OBJS))
libjulia-release: $(build_shlibdir)/libjulia.$(SHLIB_EXT)

clean:
	-rm -f $(build_shlibdir)/libjulia*
	-rm -f $(BUILDDIR)/julia_flisp.boot $(BUILDDIR)/julia_flisp.boot.inc
	-rm -f $(BUILDDIR)/*.dbg.obj $(BUILDDIR)/*.o *~ $(BUILDDIR)/*.$(SHLIB_EXT) $(BUILDDIR)/*.a *#

clean-flisp:
	-$(MAKE) -C flisp clean BUILDDIR='$(abspath $(BUILDDIR)/flisp)'

clean-support:
	-$(MAKE) -C support clean BUILDDIR='$(abspath $(BUILDDIR)/support)'


cleanall: clean clean-flisp clean-support

.PHONY: debug release clean cleanall clean-*

