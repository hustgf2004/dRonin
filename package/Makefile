# Set up a default goal
.DEFAULT_GOAL := help

# Locate the root of the tree
WHEREAMI := $(dir $(lastword $(MAKEFILE_LIST)))
ROOT_DIR := $(realpath $(WHEREAMI)/../)

# import macros common to all supported build systems
include $(ROOT_DIR)/make/system-id.mk

TOOLS_DIR := $(ROOT_DIR)/tools

# include the tools makefile
include $(ROOT_DIR)/make/tools.mk

# Set up some macros
BUILD_DIR    := $(ROOT_DIR)/build
VERSION_CMD  := $(PYTHON) $(ROOT_DIR)/make/scripts/version-info.py --path="$(ROOT_DIR)"
MACHINE_TYPE := $(shell uname -m)
ifeq ($(MACHINE_TYPE), x86_64)
PLATFORM_NAME:=amd64
else
PLATFORM_NAME:=i386
endif
PLATFORM := win
UNAME := $(shell uname)
ifeq ($(UNAME), Linux)
  PLATFORM := linux
endif
ifeq ($(UNAME), Darwin)
  PLATFORM := macos
endif
# Clean build options (recommended for package testing only)
ifeq ($(CLEAN_BUILD), NO)
CLEAN_GROUND := NO
CLEAN_FLIGHT := NO
else
CLEAN_GROUND := YES
CLEAN_FLIGHT := YES
endif

# If there is a tag, use it for label instead of date-hash string
PACKAGE_TAG  := $(shell $(VERSION_CMD) --format=\$${TAG})
ifneq ($(PACKAGE_TAG),)
PACKAGE_TAGN := $(shell $(VERSION_CMD) --format=\$${TAG}\$${DIRTY})
else
PACKAGE_TAGN := $(shell $(VERSION_CMD) --format=\$${DATE}-\$${TAG_OR_HASH8}\$${DIRTY})
endif

PACKAGE_LBL  := $(PLATFORM)_$(PACKAGE_TAGN)

PACKAGE_DIR  := $(BUILD_DIR)/package-$(PACKAGE_LBL)

FLIGHTPKGNAME := $(PACKAGE_DIR)/flight-$(PACKAGE_TAGN).zip

ifeq ($(BUILD_ALL_DEPENDENCIES), YES)
	STANDALONE_DEPENDENCIES = gcs
	EXTRACT_FIRMWARE_DEPENDENCIES = package_flight
else
	STANDALONE_DEPENDENCIES =
	EXTRACT_FIRMWARE_DEPENDENCIES =
endif

USE_MSVC ?= NO

GCS_PACKAGE_NAME:= dronin_$(PACKAGE_LBL)
GCS_PACKAGE_ROOT := $(GCS_PACKAGE_NAME)

help:
	@echo "   This Makefile is intended to be executed by the toplevel makefile"
	@echo " Type make help in the top of the dRonin tree for help."
	@echo

$(PACKAGE_DIR):
	mkdir -p $@

# Order-only dependencies

package_ground: | ground_package package_ground_symbols

package_installer: | installer_package

package_all: package_ground extract_firmware

extract_firmware: $(EXTRACT_FIRMWARE_DEPENDENCIES)
	$(V1) [ -f "$(FLIGHTPKGNAME)" ] ||  { echo Could not find the flight package file $(FLIGHTPKGNAME); exit 1; }
	@echo $@ starting
	mkdir -p "$(PACKAGE_DIR)"/firmware_$(PACKAGE_LBL)
	unzip -u -o "$(FLIGHTPKGNAME)" -d "$(PACKAGE_DIR)"/firmware_$(PACKAGE_LBL)

package_info:
	$(shell $(VERSION_CMD) --uavodir=$(ROOT_DIR)/shared/uavobjectdefinition --template=$(ROOT_DIR)/make/templates/packageversioninfo.json --outfile=$(PACKAGE_DIR)/packageversioninfo.json)

package_ground_symbols:
	@echo Gathering GCS debug symbols
	$(V1)$(PYTHON) "$(ROOT_DIR)/package/package_symbols.py" -m -d "$(DUMP_SYMBOLS_TOOL)" -p "$(PACKAGE_DIR)" "$(PACKAGE_DIR)/$(GCS_PACKAGE_ROOT)"

package_flight_symbols: extract_firmware
	$(V1)$(RM) -rf "$(PACKAGE_DIR)-symbols/flight"
	$(V1)mkdir -p "$(PACKAGE_DIR)-symbols/flight"
	$(V1)cd "$(PACKAGE_DIR)/firmware_$(PACKAGE_LBL)" && mv *.debug "$(PACKAGE_DIR)-symbols/flight"

.PHONY: help
.PHONY: package_ground package_all package_installer package_info package_ground_symbols package_flight_symbols
.PHONY: extract_firmware

ground_package: ground_package_os_specific
	@echo $@ starting
	$(V1)cd $(PACKAGE_DIR)/$(GCS_PACKAGE_ROOT) && find . \( -type f -o -type l \) -print > $(PACKAGE_DIR)/$(GCS_PACKAGE_ROOT)/filelist.lst

# Decide on a verbosity level based on the V= parameter
export AT := @

ifndef V
export V0    :=
export V1    := $(AT)
else ifeq ($(V), 0)
export V0    := $(AT)
export V1    := $(AT)
else ifeq ($(V), 1)
endif

ground_package_common: standalone
	@echo $@ starting
	$(V1) mkdir -p $(PACKAGE_DIR)/$(GCS_PACKAGE_NAME)/bin
	$(V1) mkdir -p $(PACKAGE_DIR)/$(GCS_PACKAGE_NAME)/share
	$(V1) mkdir -p $(PACKAGE_DIR)/$(GCS_PACKAGE_NAME)/lib
	$(V1) cp -v -R $(BUILD_DIR)/ground/gcs/bin $(PACKAGE_DIR)/$(GCS_PACKAGE_NAME)
	$(V1) cp -v -R $(BUILD_DIR)/ground/gcs/share $(PACKAGE_DIR)/$(GCS_PACKAGE_NAME)
	$(V1) cp -v -R $(BUILD_DIR)/ground/gcs/lib $(PACKAGE_DIR)/$(GCS_PACKAGE_NAME)
	$(V1) cp -v $(ROOT_DIR)/CREDITS.txt $(PACKAGE_DIR)/$(GCS_PACKAGE_NAME)
	$(V1) cp -v $(ROOT_DIR)/LICENSE.txt $(PACKAGE_DIR)/$(GCS_PACKAGE_NAME)
	$(V1) find $(PACKAGE_DIR)/$(GCS_PACKAGE_NAME) -type f -name "*Makefile" -exec rm -f {} \;
	$(V1) find $(PACKAGE_DIR)/$(GCS_PACKAGE_NAME) -type f -name "*qmake.cache" -exec rm -f {} \;
	$(V1) find $(PACKAGE_DIR)/$(GCS_PACKAGE_NAME) -type f -name "*udp_test" -exec rm -f {} \;
	$(V1) find $(PACKAGE_DIR)/$(GCS_PACKAGE_NAME) -type f -name "*gcsversioninfo.h" -exec rm -f {} \;

.PHONY: package_flight
package_flight: $(FLIGHTPKGNAME)

$(FLIGHTPKGNAME): $(PACKAGE_DIR) $(FW_FILES)
	$(ZIPBIN) -j --must-match $@ $(FW_FILES)

include $(WHEREAMI)/Makefile.$(PLATFORM)
