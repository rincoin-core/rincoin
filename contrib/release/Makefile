# contrib/release/Makefile — Rincoin Core multi-platform build & sign wrapper
#
# One command per target, built via the upstream `depends` system:
#
#   make linux        # x86_64 Linux, CLI-only   -> rincoin-<VERSION>-x86_64-linux-gnu.tar.gz
#   make windows      # x86_64 Windows, CLI-only -> rincoin-<VERSION>-win64.zip
#   make aarch64      # ARM64 Linux, CLI-only    -> rincoin-<VERSION>-aarch64-linux-gnu.tar.gz
#   make linux-full   # same target WITH the Qt GUI (GUI=1); also windows-full / aarch64-full
#   make all          # linux+windows+aarch64, CLI-only, serial (see .NOTPARALLEL)
#   make all-full     # the three above WITH the Qt GUI, serial
#   make sign         # SHA256SUMS over ALL artifacts + GPG detached-sign
#   make dist         # source tarball via git archive -> rincoin-<VERSION>.tar.gz
#   make release      # == make all && make dist && make sign  (full Linux-side flow)
#   make verify       # list artifacts + the §10-6 verification reminder
#   make clean        # remove staging (keeps depends/ cache)
#   make distclean    # also wipe the three depends/<host> build dirs
#   make help         # this list (default goal)
#
# Every build is captured to a timestamped log under $(LOGDIR) (default ~/logs)
# and the wrapper exits with the build's real status (bash PIPESTATUS[0]), so a
# failed compile is never masked by `tee`.
#
# Toggles (make VAR=value target):
#   GUI=1             # build the Qt GUI on a plain target (equivalent to a -full name)
#   JOBS=8            # parallel jobs INSIDE each build (default: nproc)
#   VERSION=1.1.0     # file version, no "v"   (default: parsed from configure.ac)
#   TAG=v1.1.0-rc1    # git tag / git-archive ref (default from configure.ac; RC uses -rcN)
#   KEY=<fpr>         # GPG signing key (default: the Core maintainer fingerprint below)
#   LOGDIR=<dir>      # where per-build logs are written (default: ~/logs)
#   EXTRA_CONFIG=...  # extra ./configure flags, e.g. --with-pic on R_X86_64_PC32, or
#                     #   --disable-tests to skip building test_rincoin (release binary)
#
# STATUS: cross-built binaries are EXPERIMENTAL for Rincoin. RinHash
# (BLAKE3 -> Argon2d -> SHA3-256) and BDB-4.8 cross-builds are NOT yet confirmed
# for win64/aarch64 — see release-process.md §10-6. This wrapper automates the
# *flow*; it does not by itself bless the output for release.

SHELL := /bin/bash

# Never build two targets concurrently in the same source tree (they share src/
# and we distclean between them). Each build still uses -jJOBS internally.
.NOTPARALLEL:

# Repo root = two levels up from this Makefile.
ROOT  := $(abspath $(dir $(lastword $(MAKEFILE_LIST)))../..)

JOBS  ?= $(shell nproc 2>/dev/null || echo 4)

# Version assembled from configure.ac (RIP-0001: GENERATION.MAJOR.MINOR + RC).
# Parsed with paren-free greps: a literal ')' inside a $(shell ...) would close
# the make function early, so we avoid awk field classes / sed capture groups.
V_MAJ := $(shell grep -o '_CLIENT_VERSION_MAJOR, *[0-9][0-9]*' "$(ROOT)/configure.ac" | grep -o '[0-9][0-9]*')
V_MIN := $(shell grep -o '_CLIENT_VERSION_MINOR, *[0-9][0-9]*' "$(ROOT)/configure.ac" | grep -o '[0-9][0-9]*')
V_REV := $(shell grep -o '_CLIENT_VERSION_REVISION, *[0-9][0-9]*' "$(ROOT)/configure.ac" | grep -o '[0-9][0-9]*')
V_RC  := $(shell grep -o '_CLIENT_VERSION_RC, *[0-9][0-9]*' "$(ROOT)/configure.ac" | grep -o '[0-9][0-9]*')

# The leading "v" lives ONLY on the git tag; distributed files carry the bare
# version (no "v"). RC handling is intentionally SPLIT between the two:
#   - files / package version use rcN with NO hyphen, so the version stays a
#     single token and rincoin-<ver>-<host>.tar.gz parses unambiguously; this
#     also matches `rincoind -version` and automake's PACKAGE_VERSION.
#   - the git TAG uses the readable Linux-style -rcN (hyphen).
#   release :  VERSION = 1.1.0     TAG = v1.1.0      files rincoin-1.1.0-*
#   candidate: VERSION = 1.1.0rc1  TAG = v1.1.0-rc1  files rincoin-1.1.0rc1-*
# (To make the files hyphenated too, change VERSION's rc to -rc and set
#  TAG ?= v$(VERSION).)
BASE    := $(V_MAJ).$(V_MIN).$(V_REV)
VERSION ?= $(BASE)$(if $(filter-out 0,$(V_RC)),rc$(V_RC),)
TAG     ?= v$(BASE)$(if $(filter-out 0,$(V_RC)),-rc$(V_RC),)

# Default release signing key (release-process.md §9-3).
KEY   ?= ED20B6354EE4526D01F83B538B6E3BF45C714ECA

# DESTDIR staging (one subdir per target) and the final output dir.
RELEASE ?= $(ROOT)/release
OUTDIR  ?= $(ROOT)/release-artifacts

EXTRA_CONFIG ?=

# GUI toggle: drives both the depends NO_QT switch and the ./configure flag.
GUI ?= 0
ifeq ($(GUI),1)
  DEPENDS_QT :=
  GUI_FLAG   := --with-gui=qt5
else
  DEPENDS_QT := NO_QT=1
  GUI_FLAG   := --without-gui
endif

.DEFAULT_GOAL := help

# --- per-target build template -------------------------------------------------
# $(call BUILD,<name>,<HOST triple>,<target strip program>,<default-PIC 0|1>)
#
# $(1)..$(4) and $(if ...) expand at $(eval) time (baked per target).
# $$(VAR) stays deferred and expands when the recipe runs.
define BUILD
.PHONY: $(1)
$(1):
	@echo "==> [$(1)] HOST=$(2)  VERSION=$$(VERSION)  GUI=$$(GUI)  JOBS=$$(JOBS)"
	@mkdir -p "$$(OUTDIR)"
	@echo "==> [$(1)] depends (slowest step; first run needs network)"
	$$(MAKE) -C "$$(ROOT)/depends" HOST=$(2) $$(DEPENDS_QT) -j$$(JOBS)
	@echo "==> [$(1)] autogen + configure"
	cd "$$(ROOT)" && ./autogen.sh
	cd "$$(ROOT)" && CONFIG_SITE="$$(ROOT)/depends/$(2)/share/config.site" \
	    ./configure --prefix=/ $$(GUI_FLAG) --disable-bench \
	        $(if $(filter 1,$(4)),--with-pic,) $$(EXTRA_CONFIG)
	@echo "==> [$(1)] make -j$$(JOBS)"
	$$(MAKE) -C "$$(ROOT)" -j$$(JOBS)
	@echo "==> [$(1)] install into staging"
	rm -rf "$$(RELEASE)/$(1)"
	$$(MAKE) -C "$$(ROOT)" DESTDIR="$$(RELEASE)/$(1)" install
	@echo "==> [$(1)] strip with $(3)"
	-$(3) "$$(RELEASE)/$(1)"/bin/* 2>/dev/null
	@echo "==> [$(1)] package"
	@if [ "$(2)" = "x86_64-w64-mingw32" ]; then \
	    rm -f "$$(OUTDIR)/rincoin-$$(VERSION)-win64.zip"; \
	    ( cd "$$(RELEASE)/$(1)" && zip -qr "$$(OUTDIR)/rincoin-$$(VERSION)-win64.zip" bin ); \
	 else \
	    tar -C "$$(RELEASE)/$(1)" -czf "$$(OUTDIR)/rincoin-$$(VERSION)-$(2).tar.gz" bin; \
	 fi
	@echo "==> [$(1)] distclean source tree for next target (depends/ kept)"
	-cd "$$(ROOT)" && $$(MAKE) distclean >/dev/null 2>&1
	@echo "==> [$(1)] DONE -> $$(OUTDIR)"
endef

# name       HOST triple            strip program                 PIC default
# Raw build engines (underscore = internal). Users call the logged wrappers below.
$(eval $(call BUILD,_linux,x86_64-linux-gnu,strip,1))
$(eval $(call BUILD,_windows,x86_64-w64-mingw32,x86_64-w64-mingw32-strip,0))
$(eval $(call BUILD,_aarch64,aarch64-linux-gnu,aarch64-linux-gnu-strip,0))

# --- logged public targets ---------------------------------------------------
# Each build is captured to a timestamped log under $(LOGDIR) (default ~/logs)
# and the wrapper exits with the build's real status (bash PIPESTATUS[0]), so a
# failed compile is never masked by `tee`.
#   make linux        CLI-only (respects GUI=... if set; default 0)
#   make linux-full   same target WITH the Qt GUI (GUI=1)  [also windows / aarch64]
LOGDIR ?= $(HOME)/logs

CLI_TARGETS  := linux windows aarch64
FULL_TARGETS := linux-full windows-full aarch64-full
.PHONY: $(CLI_TARGETS) $(FULL_TARGETS)

$(CLI_TARGETS):
	@mkdir -p "$(LOGDIR)"
	@log="$(LOGDIR)/$@-$$(date +%F-%H%M%S).log"; \
	 echo "==> [$@] logging to $$log"; \
	 set -o pipefail; \
	 $(MAKE) _$@ GUI=$(GUI) 2>&1 | tee "$$log"; \
	 rc=$${PIPESTATUS[0]}; \
	 echo "==> [$@] build exit: $$rc"; \
	 echo "    log: $$log"; \
	 exit $$rc

$(FULL_TARGETS):
	@mkdir -p "$(LOGDIR)"
	@log="$(LOGDIR)/$@-$$(date +%F-%H%M%S).log"; \
	 echo "==> [$@] logging to $$log"; \
	 set -o pipefail; \
	 $(MAKE) _$(patsubst %-full,%,$@) GUI=1 2>&1 | tee "$$log"; \
	 rc=$${PIPESTATUS[0]}; \
	 echo "==> [$@] build exit: $$rc"; \
	 echo "    log: $$log"; \
	 exit $$rc

.PHONY: all all-full
all:      linux windows aarch64
all-full: linux-full windows-full aarch64-full

.PHONY: sign
sign:
	@bash "$(ROOT)/contrib/release/sign.sh" "$(OUTDIR)" "$(KEY)"

# Source tarball via `git archive` — a snapshot of the signed tag, matching
# modern Bitcoin Core's source release: the generated ./configure is NOT
# included, so the builder runs ./autogen.sh first (same as a git checkout).
# Named rincoin-<VERSION>.tar.gz to match the binary artifacts, so sign.sh's glob
# (rincoin-*.tar.gz) picks it up automatically — no sign.sh change needed.
# gzip -n drops the archive's embedded timestamp for a reproducible byte stream;
# git archive already fixes entry mtimes to the commit, so the same tag yields
# the same tarball. Dry-run before tagging with:  make dist DIST_REF=HEAD
DIST_REF ?= $(TAG)

.PHONY: dist
dist:
	@mkdir -p "$(OUTDIR)"
	@git -C "$(ROOT)" rev-parse -q --verify "$(DIST_REF)^{commit}" >/dev/null || { \
	    echo "error: git ref '$(DIST_REF)' not found in $(ROOT)" >&2; \
	    echo "       tag the release first, or dry-run with: make dist DIST_REF=HEAD" >&2; \
	    exit 1; \
	 }
	@echo "==> [dist] git archive $(DIST_REF) -> rincoin-$(VERSION).tar.gz"
	@git -C "$(ROOT)" archive --format=tar --prefix="rincoin-$(VERSION)/" "$(DIST_REF)" \
	    | gzip -9n > "$(OUTDIR)/rincoin-$(VERSION).tar.gz"
	@echo "    -> $(OUTDIR)/rincoin-$(VERSION).tar.gz ($$(du -h "$(OUTDIR)/rincoin-$(VERSION).tar.gz" | cut -f1))"

.PHONY: release
release:
	@$(MAKE) all
	@$(MAKE) dist
	@$(MAKE) sign

.PHONY: verify
verify:
	@echo "Artifacts in $(OUTDIR):"
	@ls -l "$(OUTDIR)"/rincoin-$(VERSION)* 2>/dev/null || echo "  (none — run 'make all' first)"
	@echo
	@echo "Cross-built binaries are NOT release-blessed until release-process.md"
	@echo "§10-6 passes. Confirm, per target (wine / qemu-user / real hardware):"
	@echo "  - ./rincoind -version runs and reports the $(TAG) version"
	@echo "  - getblockchaininfo shows the correct consensus params (CH 840k, RIN3)"
	@echo "  - depends/<host>/lib/libdb_cxx-4.8.a exists for each host"

.PHONY: clean
clean:
	rm -rf "$(RELEASE)"
	-cd "$(ROOT)" && $(MAKE) distclean >/dev/null 2>&1

.PHONY: distclean
distclean: clean
	rm -rf "$(ROOT)/depends/x86_64-linux-gnu" \
	       "$(ROOT)/depends/x86_64-w64-mingw32" \
	       "$(ROOT)/depends/aarch64-linux-gnu"

.PHONY: help
help:
	@echo "Rincoin Core — multi-platform build & sign"
	@echo
	@echo "  make linux | windows | aarch64        CLI-only build (logged)"
	@echo "  make linux-full | windows-full | ...  build WITH the Qt GUI (logged)"
	@echo "  make all | all-full                   all three, serial (CLI | GUI)"
	@echo "  make sign                             SHA256SUMS + GPG detached-sign (all artifacts)"
	@echo "  make dist                             source tarball via git archive (rincoin-<VERSION>.tar.gz)"
	@echo "  make release                          all + dist + sign  (full Linux-side flow)"
	@echo "  make verify                           list artifacts + §10-6 checklist"
	@echo "  make clean | distclean                remove staging [ + depends build dirs ]"
	@echo
	@echo "  each build tees to LOGDIR/<target>-<timestamp>.log (LOGDIR shown below)"
	@echo "  toggles: GUI=1  JOBS=N  VERSION=X.Y.Z  KEY=<fpr>  LOGDIR=<dir>  EXTRA_CONFIG='--with-pic'"
	@echo
	@echo "  ROOT   = $(ROOT)"
	@echo "  VERSION= $(VERSION)   (files, no v)      TAG = $(TAG)   (git tag)"
	@echo "  LOGDIR = $(LOGDIR)"
	@echo "  OUTDIR = $(OUTDIR)"
