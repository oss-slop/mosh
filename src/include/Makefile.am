BUILT_SOURCES = version.h
nodist_INCLUDES = version.h

clean-local:
	@rm -rf version.h

version.h:	../../VERSION.stamp
	@test -s $<
	@set -e; \
	git_commit="$$(cd $(top_srcdir) && ( command -v git >/dev/null 2>&1 && git rev-parse --short=12 HEAD 2>/dev/null ) || printf unknown)"; \
	printf '#define BUILD_VERSION "%s"\n' "$$(cat $<)" > $@.new; \
	printf '#define BUILD_GIT_COMMIT "%s"\n' "$$git_commit" >> $@.new
	@set -e; if ! diff -q $@ $@.new > /dev/null 2>&1; then \
		mv -f $@.new $@; \
	fi
	@rm -f $@.new
