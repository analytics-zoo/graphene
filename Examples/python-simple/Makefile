# Use one of the following commands to build the manifest for Python3:
#
# - make                Building for Linux
# - make DEBUG=1        Building for Linux (with Graphene debug output)
# - make SGX=1          Building for SGX
# - make SGX=1 DEBUG=1  Building for SGX (with Graphene debug output)
#
# Use `make clean` to remove Graphene-generated files.

include ../../Scripts/Makefile.configs

# Python constants are declared in Makefile.python
include ../../Scripts/Makefile.python

SGX_SIGNER_KEY ?= ../../Pal/src/host/Linux-SGX/signer/enclave-key.pem

RUNTIMEDIR = $(PREFIX)/lib/graphene/sgx/runtime

ifeq ($(DEBUG),1)
GRAPHENEDEBUG = inline
else
GRAPHENEDEBUG = none
endif

.PHONY: all
all: python.manifest | python
ifeq ($(SGX),1)
all: python.manifest.sgx python.token python.sig
endif

# Python dependencies (generate from ldd):
#
# For SGX, the manifest needs to list all the libraries loaded during the
# execution, so that the signer can include the file checksums.
#
# The dependencies are generated from the ldd results.

# We need to replace Glibc dependencies with Graphene-specific Glibc. The Glibc
# binaries are already listed in the manifest template, so we can skip them
# from the ldd results.
GLIBC_DEPS = linux-vdso /lib64/ld-linux-x86-64 libc libm librt libdl libutil libpthread

# Define the python libraries which are dynamically loaded.
PY_LIBS = \
	$(PYTHON_STDLIB)/lib-dynload/_hashlib$(PYTHON_EXT_SUFFIX) \
	$(PYTHON_STDLIB)/lib-dynload/_ctypes$(PYTHON_EXT_SUFFIX) \
	$(PYTHON_STDLIB)/lib-dynload/_ssl$(PYTHON_EXT_SUFFIX) \
	$(PYTHON_STDLIB)/lib-dynload/_bz2$(PYTHON_EXT_SUFFIX) \
	$(PYTHON_STDLIB)/lib-dynload/_lzma$(PYTHON_EXT_SUFFIX) \
	$(PYTHON_STDLIB)/lib-dynload/_json$(PYTHON_EXT_SUFFIX)

ifeq ($(SGX),1)
PY_LIBS += $(PYTHON_DISTLIB)/apt_pkg$(PYTHON_EXT_SUFFIX)
endif

define PY_LIBS_TRUSTED_LIBS
sgx.trusted_files.hashlib = "file:$(PYTHON_STDLIB)/lib-dynload/_hashlib$(PYTHON_EXT_SUFFIX)"
sgx.trusted_files.ctypes = "file:$(PYTHON_STDLIB)/lib-dynload/_ctypes$(PYTHON_EXT_SUFFIX)"
sgx.trusted_files.ssl = "file:$(PYTHON_STDLIB)/lib-dynload/_ssl$(PYTHON_EXT_SUFFIX)"
sgx.trusted_files.bz2 = "file:$(PYTHON_STDLIB)/lib-dynload/_bz2$(PYTHON_EXT_SUFFIX)"
sgx.trusted_files.lzma = "file:$(PYTHON_STDLIB)/lib-dynload/_lzma$(PYTHON_EXT_SUFFIX)"
sgx.trusted_files.json = "file:$(PYTHON_STDLIB)/lib-dynload/_json$(PYTHON_EXT_SUFFIX)"
sgx.trusted_files.aptpkg = "file:$(PYTHON_DISTLIB)/apt_pkg$(PYTHON_EXT_SUFFIX)"

endef

ifeq ($(PYTHONSHORTVERSION),35)
define PYTHON_TRUSTED_SCRIPTS
sgx.trusted_files.python21 = "file:$(PYTHON_STDLIB)/_sysconfigdata.py"
sgx.trusted_files.python22 = "file:$(PYTHON_STDLIB)/plat-$(PYTHON_MULTIARCH)/_sysconfigdata_m.py"

endef
else
define PYTHON_TRUSTED_SCRIPTS
sgx.trusted_files.python21 = "file:$(PYTHON_STDLIB)/_sysconfigdata_m_linux_$(PYTHON_MULTIARCH).py"

endef
endif

# Listing all the Python dependencies, besides Glibc libraries
.INTERMEDIATE: python-ldd
python-ldd:
	@for F in $(PY_LIBS); do ldd $$F >> $@ || exit 1; done

.INTERMEDIATE: python-deps
python-deps: python-ldd
	@cat $< | awk '{if ($$2 =="=>") {split($$1,s,/\./); print s[1]}}' \
		| sort | uniq | grep -v -x $(patsubst %,-e %,$(GLIBC_DEPS)) > $@

# Generating manifest rules for Python dependencies
.INTERMEDIATE: python-trusted-libs
.ONESHELL:
python-trusted-libs: python-deps
	@PY_LIBS="$(PY_LIBS)" && \
	for F in `cat python-deps`; do \
		P=`ldd $$PY_LIBS | grep $$F | awk '{print $$3; exit}'`; \
		N=`echo $$F | tr --delete '+-'`; \
		printf 'sgx.trusted_files.%s = "file:%s"\n' "$$N" "$$P"; \
	done > $@
	printf %s '$(PY_LIBS_TRUSTED_LIBS)' >> $@

.INTERMEDIATE: python-trusted-scripts
.ONESHELL:
python-trusted-scripts:
	true
	printf %s '$(PYTHON_TRUSTED_SCRIPTS)' >> $@

python.manifest: python.manifest.template python-trusted-libs python-trusted-scripts
	sed \
		-e 's|$$(RUNTIMEDIR)|'"$(RUNTIMEDIR)"'|g' \
		-e 's|$$(GRAPHENEDEBUG)|'"$(GRAPHENEDEBUG)"'|g' \
		-e 's|$$(PYTHON_STDLIB)|'"$(PYTHON_STDLIB)"'|g' \
		-e 's|$$(PYTHON_DISTLIB)|'"$(PYTHON_DISTLIB)"'|g' \
		-e 's|$$(PYTHON_MULTIARCH)|'"$(PYTHON_MULTIARCH)"'|g' \
		-e 's|$$(PYTHONEXEC)|'"$(PYTHONEXEC)"'|g' \
		-e 's|$$(ARCH_LIBDIR)|'"$(ARCH_LIBDIR)"'|g' \
		-e 's|$$(ARCH_LONG)|'"$(ARCH_LONG)"'|g' \
		$< > $@
	cat python-trusted-scripts python-trusted-libs >> $@

# Python manifests for SGX:
#   Generating the SGX-specific manifest (python.manifest.sgx), the enclave signature,
#   and the token for enclave initialization.

python.manifest.sgx: python.manifest
	graphene-sgx-sign \
		-exec python \
		-libpal $(RUNTIMEDIR)/libpal-Linux-SGX.so \
		-key $(SGX_SIGNER_KEY) \
		-manifest $< -output $@

python.sig: python.manifest.sgx

python.token: python.sig
	graphene-sgx-get-token -output $@ -sig $<

python:
	ln -s $(PYTHONEXEC) $@

.PHONY: check
check: all
	./run-tests.sh > OUTPUT_TEST 2> /dev/null
	@grep -q "Success 1/3" OUTPUT_TEST
	@grep -q "Success 2/3" OUTPUT_TEST
	@grep -q "Success 3/3" OUTPUT_TEST
	@rm OUTPUT_TEST

.PHONY: clean
clean:
	$(RM) *.manifest *.manifest.sgx *.token *.sig python OUTPUT* *.PID
	$(RM) -r scripts/__pycache__

.PHONY: distclean
distclean: clean
