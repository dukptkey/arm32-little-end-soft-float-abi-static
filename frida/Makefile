FRIDA_VERSION ?= 17.19.0
HOST_TRIPLET  ?= arm-linux-gnueabi
FRIDA_HOST    ?= $(HOST_TRIPLET)

WORK    := $(dir $(abspath $(firstword $(MAKEFILE_LIST))))
WORK    := $(WORK:/=)
ROOT    := $(abspath $(WORK)/..)
SRC     := $(WORK)/src
BUILD   := $(WORK)/build
OUT     := $(ROOT)/frida-server-$(FRIDA_VERSION)-arm32-static

CROSS_CC := $(HOST_TRIPLET)-gcc
STRIP    := $(HOST_TRIPLET)-strip

.PHONY: all fetch deps configure build install check clean distclean

all: install

fetch: $(WORK)/.patched

$(SRC)/configure:
	git ls-remote --exit-code --tags https://github.com/frida/frida $(FRIDA_VERSION) >/dev/null
	mkdir -p $(WORK)
	git clone --depth 1 --branch $(FRIDA_VERSION) https://github.com/frida/frida $(SRC)
	cd $(SRC) && git submodule update --init --depth 1 --recursive

# Frida's "arm" host is the soft-float EABI (arm-linux-gnueabi) but targets ARMv5T
# and links dynamically. Retarget it to ARMv7-A and link frida-server statically
# (only the executable: shared modules such as glib-networking cannot take -static).
# glib's stub libintl and glibc's libc.a both define _nl_msg_cat_cntr, hence
# --allow-multiple-definition. GCC 15 also rejects an incompatible-pointer-types bug in OpenSSL's armcap.c.
$(WORK)/.patched: $(SRC)/configure
	cd $(SRC) && git submodule update --init --depth 1 releng
	sed -i 's/"-march=armv5t"/"-march=armv7-a"/; s/linker_flags += \["-static-libgcc"\]/linker_flags += ["-static-libgcc", "-Wl,--allow-multiple-definition"]/; s/^\( *\)"-ffunction-sections",/&\n\1"-Wno-error=incompatible-pointer-types",/' $(SRC)/releng/env_generic.py
	grep -q 'armv7-a' $(SRC)/releng/env_generic.py
	grep -q 'Wno-error=incompatible-pointer-types' $(SRC)/releng/env_generic.py
	touch $@

# No prebuilt SDK is published for linux-arm, so build the dependencies
# (glib, openssl, ...) from source with the patched flags. V8 is skipped: its
# snapshot needs a 32-bit build machine. The server uses the QuickJS runtime.
# OpenSSL's pkg-config files say -latomic, which Meson resolves to the shared
# library; name the static archive so the server can link with -static.
deps: $(WORK)/.deps

$(WORK)/.deps: $(WORK)/.patched
	command -v $(CROSS_CC) >/dev/null
	cd $(SRC) && python3 -I releng/deps.py build --bundle=sdk --host=$(FRIDA_HOST) --exclude=v8
	sed -i 's/-latomic/-l:libatomic.a/' $(SRC)/deps/sdk-linux-arm/lib/pkgconfig/lib*.pc
	grep -q -- '-l:libatomic.a' $(SRC)/deps/sdk-linux-arm/lib/pkgconfig/libcrypto.pc
	touch $@

configure: $(BUILD)/build.ninja

# Frida configures into the current directory, which must not contain the sources.
# The server executable (not the agent or other shared modules) is linked with -static.
$(BUILD)/build.ninja: $(WORK)/.deps
	cd $(SRC) && python3 -I tools/ensure-submodules.py
	sed -i "s/^extra_link_args = \[\]/extra_link_args = ['-static']/" $(SRC)/subprojects/frida-core/server/meson.build
	grep -q "extra_link_args = \['-static'\]" $(SRC)/subprojects/frida-core/server/meson.build
	mkdir -p $(BUILD)
	cd $(BUILD) && $(SRC)/configure \
		--host=$(HOST_TRIPLET) \
		--prefix=$(WORK)/prefix \
		-- \
		-Dserver=enabled \
		-Dfrida-gum:v8=disabled \
		-Dgadget=disabled \
		-Dinject=disabled \
		-Dfrida_tools=disabled \
		-Dgraft_tool=disabled

build: $(BUILD)/build.ninja
	ninja -C $(BUILD)

install: build
	cp $(BUILD)/subprojects/frida-core/server/frida-server $(OUT)
	$(STRIP) $(OUT)
	chmod +x $(OUT)
	file $(OUT)

check: $(OUT)
	file $(OUT) | grep -q 'statically linked'
	readelf -A $(OUT) | grep -E 'Tag_CPU_arch|Tag_ABI_VFP_args|Tag_FP_arch' || true
	qemu-arm-static $(OUT) --version

clean:
	mkdir -p $(ROOT)/.trash
	[ ! -d $(BUILD) ] || mv $(BUILD) $(ROOT)/.trash/build-$$(date +%s)

distclean:
	mkdir -p $(ROOT)/.trash
	for p in $(SRC) $(BUILD) $(WORK)/.patched $(WORK)/.deps $(WORK)/build.log \
		$(WORK)/frida-*.txt $(WORK)/frida-*.py; do \
		[ ! -e $$p ] || mv $$p $(ROOT)/.trash/$$(basename $$p)-$$(date +%s); \
	done
