##
# Makefile for python
##

Project               = python
Extra_Configure_Flags = --enable-ipv6 --with-system-expat --with-threads --enable-framework=/System/Library/Frameworks --enable-toolbox-glue --with-system-ffi --with-gcc=$(WITH_GCC) CC=$(WITH_GCC) CXX=$(WITH_GXX)
# Define the environment variable DEBUG to YES, to build without optimization
ifeq ($(DEBUG),YES)
Extra_Configure_Flags += --with-pydebug
endif

##---------------------------------------------------------------------
# Extra_CC_Flags and Extra_LD_Flags are needed because CFLAGS gets overridden
# by the RC_* variables.  These values would normally be set by the default
# python Makefile
#
# Workaround for 3281234 (test_coercion failure due to non IEEE-754 in
# optimizer): add -mno-fused-madd flag
##---------------------------------------------------------------------
Extra_CC_Flags += -fno-common -fno-strict-aliasing -fwrapv -mno-fused-madd -DENABLE_DTRACE -DMACOSX -DNDEBUG -Wall -Wstrict-prototypes -Wshorten-64-to-32
Extra_LD_Flags += -Wl,-F.
Extra_Install_Flags   = DESTDIR='$(DSTROOT)'
GnuAfterInstall       = fixup-after-install install-plist
Extra_Environment     = CCSHARED='$(RC_CFLAGS)'

# It's a GNU Source project
include $(MAKEFILEPATH)/CoreOS/ReleaseControl/GNUSource.make

Install_Flags := $(shell echo $(Install_Flags) | sed 's/prefix=[^ ]* *//')
Install_Target = frameworkinstall
FIX = '$(SRCROOT)/fix'

##---------------------------------------------------------------------
# Patch Makefiles and pyconfig.h just after running configure
##---------------------------------------------------------------------
ConfigStamp2 = $(ConfigStamp)2

configure:: $(ConfigStamp2)

$(ConfigStamp2): $(ConfigStamp)
	ed - '$(OBJROOT)/Makefile' < $(FIX)/Makefile.ed
	ed - '$(OBJROOT)/pyconfig.h' < $(FIX)/pyconfig.ed
	$(TOUCH) $(ConfigStamp2)

##---------------------------------------------------------------------
# Fixup a lot of problems after the install
##---------------------------------------------------------------------
APPS = /Applications
USRBIN = /usr/bin
PYTHONAPPNAME = Python.app
PYTHONLAUNCHERNAME = Python Launcher.app
FRAMEWORKS = /System/Library/Frameworks
PYFRAMEWORK = $(FRAMEWORKS)/Python.framework
VERSIONSVERS = $(PYFRAMEWORK)/Versions/$(VERS)
RESOURCESVERS = $(VERSIONSVERS)/Resources
LIBPYTHONVERS = $(VERSIONSVERS)/lib/python$(VERS)
PYTHONAPP = $(RESOURCESVERS)/$(PYTHONAPPNAME)
PACONTENTS = $(PYTHONAPP)/Contents
PAMACOS = $(PACONTENTS)/MacOS
PYTHONVERS = $(APPS)/Python $(VERS)
PYTHONLAUNCHER = $(RESOURCESVERS)/$(PYTHONLAUNCHERNAME)
PLCONTENTS = $(PYTHONLAUNCHER)/Contents
RUNPYTHON = DYLD_FRAMEWORK_PATH='$(OBJROOT)' '$(OBJROOT)/python.exe'
BYTE2UTF16 = $(RUNPYTHON) $(FIX)/byte2utf16.py
UTF162BYTE = $(RUNPYTHON) $(FIX)/utf162byte.py

fixup-after-install: delete-stuff \
		     move-things-around \
		     strip-installed-files \
		     fix-PInfo \
		     fix-paths \
		     fix-usr-local-bin \
		     fix-usr-bin \
		     fix-permissions \
		     fix-config-Makefile \
		     additional-man-pages

delete-stuff:
	rm -rf '$(DSTROOT)/usr/local'

# We leave "Build Applet.app" in $(APPS) to be deleted, because it uses
# Carbon, and doesn't work in 64-bit.
move-things-around:
	mv -f '$(DSTROOT)$(PYTHONVERS)/$(PYTHONLAUNCHERNAME)' '$(DSTROOT)$(RESOURCESVERS)'
	rm -rf '$(DSTROOT)$(APPS)'

Python_Launcher = $(PLCONTENTS)/MacOS/Python Launcher

strip-installed-files:
	$(CP) $(OBJROOT)/python.exe '$(SYMROOT)'
	strip -x $(DSTROOT)$(PAMACOS)/Python
	$(CP) '$(DSTROOT)$(VERSIONSVERS)/Python' '$(SYMROOT)'
	strip -x '$(DSTROOT)$(VERSIONSVERS)/Python'
	strip -x '$(DSTROOT)$(Python_Launcher)'
	ditto '$(DSTROOT)$(LIBPYTHONVERS)'/lib-dynload/*.so '$(SYMROOT)/lib-dynload/'
	cd '$(SYMROOT)/lib-dynload' && \
	for so in *.so; do \
	    $(DSYMUTIL) $$so || exit 1; \
	done
	strip -x '$(DSTROOT)$(LIBPYTHONVERS)'/lib-dynload/*.so

fix-PInfo:
	ed - '$(DSTROOT)$(PACONTENTS)/Info.plist' < $(FIX)/pinfo.ed
	ed - '$(DSTROOT)$(PLCONTENTS)/Info.plist' < $(FIX)/pinfo.ed
	ed - '$(DSTROOT)$(RESOURCESVERS)/Info.plist' < $(FIX)/pinfo.ed

##---------------------------------------------------------------------
# As of python 2.7, the real man page is now python2.7.1 (not python.1).
##---------------------------------------------------------------------
MAN1 = /usr/share/man/man1
additional-man-pages:
	mv -f $(DSTROOT)$(MAN1)/python$(VERS).1 $(DSTROOT)$(MAN1)/python.1
	install -m 0644 $(FIX)/pydoc.1 '$(DSTROOT)$(MAN1)'
	install -m 0644 $(FIX)/pythonw.1 '$(DSTROOT)$(MAN1)'
	ln -sf pydoc.1 '$(DSTROOT)$(MAN1)/pydoc$(VERS).1'
	ln -sf python.1 '$(DSTROOT)$(MAN1)/python$(VERS).1'
	ln -sf pythonw.1 '$(DSTROOT)$(MAN1)/pythonw$(VERS).1'

PYDOC = $(USRBIN)/pydoc
PYDOCORIG = $(VERSIONSVERS)/bin/pydoc

##---------------------------------------------------------------------
# The scripts exec $(VERSIONSVERS)/bin/python$(VERS), which execs
# $(PAMACOS)/Python.  We skip the extra exec by changing the scripts
# to exec $(PAMACOS)/Python directly.
##---------------------------------------------------------------------
fix-paths:
	@set -x && \
	cd '$(DSTROOT)$(VERSIONSVERS)/bin' && \
	for i in `find . -type f | sed 's,^\./,,'`; do \
	    if [ -n "`file $$i | fgrep script`" ]; then \
		ed - $$i < $(FIX)/skipextraexec.ed; \
	    fi || exit 1; \
	done

CGIPY = $(LIBPYTHONVERS)/cgi.py
fix-usr-local-bin:
	@set -x && \
	cd '$(DSTROOT)$(VERSIONSVERS)' && \
	patch -p0 < $(FIX)/usrlocalbin.patch && \
	$(RUNPYTHON) -c "from py_compile import compile;compile('$(DSTROOT)$(CGIPY)', dfile='$(CGIPY)', doraise=True)" && \
	$(RUNPYTHON) -O -c "from py_compile import compile;compile('$(DSTROOT)$(CGIPY)', dfile='$(CGIPY)', doraise=True)"

##---------------------------------------------------------------------
# config/Makefile needs the following changes:
# remove -arch xxx flags
# 4144521 - correct LINKFORSHARED
# 3488297 - point BINDIR to /usr/local/bin
##---------------------------------------------------------------------
INSTALLPY = $(LIBPYTHONVERS)/distutils/command/install.py
fix-config-Makefile:
	ed - '$(DSTROOT)$(LIBPYTHONVERS)/config/Makefile' < $(FIX)/config_Makefile.ed

fix-usr-bin:
	@set -x && \
	cd '$(DSTROOT)$(USRBIN)' && \
	rm -f idle* && \
	for i in *; do \
	    rm -f $$i && \
	    ln -s ../..$(VERSIONSVERS)/bin/$$i || exit 1; \
	done

LIBRARYPYTHON = /Library/Python
LIBRARYPYTHONVERS = $(LIBRARYPYTHON)/$(VERS)

fix-permissions:
ifeq ($(shell id -u), 0)
	@set -x && \
	for i in Applications Developer Library; do \
	    chgrp -Rf admin $(DSTROOT)/$$i && \
	    chmod -Rf g+w $(DSTROOT)/$$i; \
	done
endif

OSV = /usr/local/OpenSourceVersions
OSL = /usr/local/OpenSourceLicenses

install-plist:
	$(MKDIR) '$(DSTROOT)$(OSV)'
	$(INSTALL_FILE) '$(SRCROOT)/$(Project).plist' '$(DSTROOT)$(OSV)/$(Project).plist'
	$(MKDIR) '$(DSTROOT)$(OSL)'
	$(INSTALL_FILE) '$(OBJROOT)/LICENSE' '$(DSTROOT)$(OSL)/$(Project).txt'