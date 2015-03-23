---
layout: post
title:  "The baseline Makefile"
keywords: Makefile make Unix Linux C++ C
published: true
categories: programming
---

Any non trivial C++ or C program requires some kind of build system. There 
are many of them available: cmake, waf, scons, and many others. You can even
roll your own. Problem is that you need to learn the "language" of that build 
system, its rules, conventions and so one. But clearly, you usally don't want
to spend that much time on that task in the beginning of your project and you 
would like focus on your software first. On the other hand getting a properly 
working and correct build system is necessary as I already said for any non 
trivial system. In this post I present a simple Makefile. You can use it 
almost as is when you start coding you C or C++ program. The most important 
feature is that it 
[automatically generates the dependencies](http://make.mad-scientist.net/papers/advanced-auto-dependency-generation/) between your 
source files. So why make (or gmake actually) and not any other build tool?
Make is huge and is not simple at all. Well, it is widely used (Android build 
system is entirely build on Makefiles for example) and widely available even 
on Windows. You can of course read a "tutorial" on make but most of the time 
those tutorials on scratch the surface of the art of creating a __correct__ 
Makefile. Ok now here is the Makefile: Please read the inline comments to 
understand how it structured.

{% highlight make %}

# Added the libraries you need if any to link to you PROGRAM
BOOST_LDFLAGS = -lboost_system
ZMQ_LDFLAGS = -lzmq
LDFLAGS = $(BOOST_LDFLAGS) $(ZMQ_LDFLAGS)
CXXFLAGS = -std=c++11
CFLAGS = 
CPPFLAGS = 
PROGRAM = htparser
OBJDIR = .objs
DEPSDIR = .deps
BUILD_DIRS = $(OBJDIR) $(DEPSDIR)
depfile = $(DEPSDIR)/$(*F)

# Invoke the c++ compiler to generate a dependency file 
# along with the object file for the source file being 
# compiled (-MMD option)
MAKEDEPEND = $(CXX) $(CPPFLAGS) $(CXXFLAGS) -MMD -o $@ -c $<
# Same thing but for c compiler
CMAKEDEPEND = $(CC) $(CPPFLAGS) $(CFLAGS) -MMD -o $@ -c $<

# List the source files that make up you program here
# or use a wildcard pattern
CSRCS = $(wildcard *.c)
# or list files explicitly as in this example:
SRCS = reply.cpp htparser.cpp request_parser.cpp
OBJS_TMP = $(SRCS:.cpp=.o)
OBJS_TMP += $(CSRCS:.c=.o)
# The object files will be placed in OBJDIR
OBJS = $(patsubst %,$(OBJDIR)/%,$(OBJS_TMP))

.PHONY: all
all: build_dirs $(PROGRAM)

# This target serves for "debugging" and shows the list of 
# objects that make up our PROGRAM
.PHONY: show_objs
show_objs:
@echo $(OBJS);

build_dirs: $(BUILD_DIRS)

$(BUILD_DIRS):
@mkdir -p $(BUILD_DIRS)

$(PROGRAM): $(OBJS)
@echo Linking $@
$(CXX) $(LDFLAGS) -o $@ $^

# Have a look at this page 
# http://make.mad-scientist.net/papers/advanced-auto-dependency-generation/
# to understand what the recipes under the pattern rules do and why
# they are necessary beyond creating the dependency. If the sed command 
# looks like ancient chinese to you; don't worry because it does.

# For c++ files
$(OBJDIR)/%.o: %.cpp
$(MAKEDEPEND)
@cat $(OBJDIR)/$*.d > $(depfile).P; \
sed -e 's/#.*//' -e 's/^[^:]*: *//' -e 's/ *\\$$//' \
-e '/^$$/ d' -e 's/$$/:/' < $(OBJDIR)/$*.d >> $(depfile).P; \
rm -f $(OBJDIR)/$*.d

# and c files
$(OBJDIR)/%.o: %.c
$(CMAKEDEPEND)
@cat $(OBJDIR)/$*.d > $(depfile).P; \
sed -e 's/#.*//' -e 's/^[^:]*: *//' -e 's/ *\\$$//' \
-e '/^$$/ d' -e 's/$$/:/' < $(OBJDIR)/$*.d >> $(depfile).P; \
rm -f $(OBJDIR)/$*.d

.PHONY: clean
clean:
rm -rf $(BUILD_DIRS) $(PROGRAM)
@echo Directory clean

# Include the dependency files if they exist.
# The -include fails silently if they don't and this is
# what we want. They will exist next time make is invoked

-include $(SRCS:%.cpp=$(DEPSDIR)/%.P)
-include $(CSRCS:%.c=$(DEPSDIR)/%.P)
{% endhighlight %}

I am not a make guru, so please post a comment if you have a suggestion or if you see something wrong
with this Makefile.



