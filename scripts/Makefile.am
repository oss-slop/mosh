EXTRA_DIST = wrap-compiler-for-flag-check mosh.pl
if BUILD_CLIENT
  bin_SCRIPTS = mosh
endif
CLEANFILES = $(bin_SCRIPTS)

mosh:	mosh.pl ../VERSION.stamp Makefile
	perl -Mdiagnostics -c $(srcdir)/mosh.pl
	@set -e; \
	git_commit="$$(cd $(top_srcdir) && ( command -v git >/dev/null 2>&1 && git rev-parse --short=12 HEAD 2>/dev/null ) || printf unknown)"; \
	sed -e "s/\@VERSION\@/`cat ../VERSION.stamp`/" \
		-e "s/\@PACKAGE_STRING\@/@PACKAGE_STRING@/" \
		-e "s/\@GIT_COMMIT\@/$$git_commit/" \
		$(srcdir)/mosh.pl > mosh
	@chmod a+x mosh
