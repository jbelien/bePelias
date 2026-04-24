# Makefile for preparing the BEST data files, to be run inside the dataprep container.
# Usage:
#  - make all REGION=all|bru|wal|vlg # runs the full pipeline for the specified region(s) and cleans intermediate files
#  - make prepare REGION=all|bru|wal|vlg # runs the full pipeline for the specified region(s) but keeps intermediate files
#  - make all_lowdisk # runs the full pipeline and delete intermediate files as soon as possible, to minimize disk usage
#  - make clean-input # cleans intermediate files from the input directory


SHELL := /usr/bin/env bash
.SHELLFLAGS := -euo pipefail -c
.DEFAULT_GOAL := help

DATA_IN := /data/in
DATA_OUT := /data

GLOBAL_ZIP := $(DATA_IN)/best-full-latest.zip
CONVERTER_JAR := /best-tools/java/converter/target/converter-1.*.jar
PREPARE_PY := /bepelias/prepare_best_files.py
PYTHON := python3

REGION ?= all

ifeq ($(REGION),all)
  REGIONS := Brussels Wallonia Flanders
  REGION_GLOB := *
  REG := 
else ifeq ($(REGION),bru)
  REGIONS := Brussels
  REG := Brussels
  REGION_GLOB := bru
else ifeq ($(REGION),wal)
  REGIONS := Wallonia
  REG := Wallonia
  REGION_GLOB := wal
else ifeq ($(REGION),vlg)
  REGIONS := Flanders
  REG := Flanders
  REGION_GLOB := vlg
else
  $(error REGION must be one of: all bru wal vlg)
endif

REGION_FLAGS_Brussels := -B
REGION_FLAGS_Wallonia := -W
REGION_FLAGS_Flanders := -F

# Markers for each stage
UNZIP_MARKERS := $(addprefix $(DATA_IN)/,$(addsuffix .regional_zip_extracted,$(REGIONS)))
XML_MARKERS := $(addprefix $(DATA_IN)/,$(addsuffix .xml_extracted,$(REGIONS)))
OA_MARKERS := $(addprefix $(DATA_IN)/,$(addsuffix .openaddress_ready,$(REGIONS)))
PREPARED_MARKER := $(DATA_OUT)/.prepared-$(REGION)

.PHONY: help all prepare \
        download unzip xml convert \
        clean clean-input clean-output \
        clean-unzip clean-xml clean-convert

help:
	@echo "Targets:"
	@echo "  make -f /bepelias/Makefile.prepare_csv download"
	@echo "  make -f /bepelias/Makefile.prepare_csv unzip REGION=all|bru|wal|vlg"
	@echo "  make -f /bepelias/Makefile.prepare_csv xml REGION=all|bru|wal|vlg"
	@echo "  make -f /bepelias/Makefile.prepare_csv convert REGION=all|bru|wal|vlg"
	@echo "  make -f /bepelias/Makefile.prepare_csv prepare REGION=all|bru|wal|vlg"
	@echo "  make -f /bepelias/Makefile.prepare_csv clean"

all: prepare clean-input

all_lowdisk: download unzip xml clean-zip convert clean-xml prepare clean-convert

download: $(GLOBAL_ZIP)

unzip: $(UNZIP_MARKERS)

xml: $(XML_MARKERS)

convert: $(OA_MARKERS)

prepare: $(PREPARED_MARKER)

# --- Step 0: download global archive ---
# Inputs: - 
# Outputs: best-full-latest.zip
$(GLOBAL_ZIP):
	mkdir -p $(DATA_IN)
	wget --progress=dot:giga -O "$@" https://opendata.bosa.be/download/best/best-full-latest.zip


# --- Step 1: extract regional zip(s) from the global zip ---
# Inputs: best-full-latest.zip
# Outputs: (Brussels|Flanders|Wallonia)(Municipality|PostalInfo|StreetName|Address)*.zip
# Marker: (Brussels|Flanders|Wallonia).regional_zip_extracted

$(DATA_IN)/%.regional_zip_extracted: $(GLOBAL_ZIP)
	mkdir -p $(DATA_IN)

	cd $(DATA_IN) && unzip -o best-full-latest.zip  $(REG)'*.zip'
	touch $@

# --- Step 2: extract XML from the regional zip(s) ---
# Input: (Brussels|Flanders|Wallonia)(Municipality|PostalInfo|StreetName|Address)*.zip
# Output: (Brussels|Flanders|Wallonia)(Municipality|PostalInfo|StreetName|Address)*.xml
# Marker: (Brussels|Flanders|Wallonia).xml_extracted

$(DATA_IN)/%.xml_extracted: $(DATA_IN)/%.regional_zip_extracted
	cd $(DATA_IN) && unzip -o $(REG)'*.zip'
	touch $@


# --- Step 3: convert XML to openaddress CSV using Java converter ---
# Inputs: (Brussels|Flanders|Wallonia)(Municipality|PostalInfo|StreetName|Address)*.xml
# Ouputs: 
#  - (Brussels|Flanders|Wallonia)_addresses.csv
#  - (Brussels|Flanders|Wallonia)_empty_street.csv
#  - (Brussels|Flanders|Wallonia)_municipalities.csv  (*)
#  - (Brussels|Flanders|Wallonia)_postal_street.csv   (*)
#  - (Brussels|Flanders|Wallonia)_postalinfo.csv      (*)
#  - (Brussels|Flanders|Wallonia)_streetnames.csv     (*)
#  - openaddress-be(bru|vlg|wal).csv                  (*)
# (*) Unused files, can be removed right away
# Marker: (Brussels|Flanders|Wallonia).openaddress_ready 

$(DATA_IN)/%.openaddress_ready: $(DATA_IN)/%.xml_extracted
	echo ".level= WARNING" > logging.properties
	 
	$$JAVA_HOME/bin/java -Djava.util.logging.config.file="logging.properties" -jar $(CONVERTER_JAR) -i $(DATA_IN) $(REGION_FLAGS_$*)
	rm -f $(DATA_IN)/openaddress-be*.csv \
	      $(DATA_IN)/*_municipalities.csv \
	      $(DATA_IN)/*_postal_street.csv \
	      $(DATA_IN)/*_postalinfo.csv \
	      $(DATA_IN)/*_streetnames.csv
	touch $@

# --- Step 4: convert openaddress CSVs into final bestaddresses CSVs ---
# Inputs: 
#  - (Brussels|Flanders|Wallonia)_addresses.csv
#  - (Brussels|Flanders|Wallonia)_empty_street.csv
# Outputs: 
#  - bestaddresses_be(bru|vlg|wal).csv
#  - bestaddresses_streets_be(bru|vlg|wal).csv
#  - bestaddresses_localities_be(bru|vlg|wal).csv
#  - bestaddresses_interpolation_be(bru|vlg|wal).csv
# Marker: .prepared-(bru|vlg|wal)

$(PREPARED_MARKER): $(OA_MARKERS)

	. /bepelias_venv/bin/activate && \
	$(PYTHON) $(PREPARE_PY) -i $(DATA_IN) -o $(DATA_OUT) -r $(REGION)
	touch $@

# --- Cleanup ---

clean: clean-input clean-output

clean-input: clean-zip clean-xml clean-convert

clean-output:
	rm -f $(DATA_OUT)/bestaddresses_*be$(REGION_GLOB).csv \
	      $(DATA_OUT)/.prepared-*

clean-zip: 
	rm -f $(DATA_IN)/*.regional_zip_extracted $(DATA_IN)/*.zip

clean-xml:
	rm -f $(DATA_IN)/*.xml_extracted $(DATA_IN)/*.xml

clean-convert:
	rm -f $(DATA_IN)/*.openaddress_ready \
	      $(DATA_IN)/Brussels*.csv \
	      $(DATA_IN)/Wallonia*.csv \
	      $(DATA_IN)/Flanders*.csv