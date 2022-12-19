.PHONY: all clean build

all:
	@echo "make foo, bar or baz!"


foo:
	@echo making foo
	@echo cqfd >foo

bar:
	@echo making bar
	@echo cqfd >bar

baz:
	@echo making baz
	@echo cqfd >baz

build:
	@mkdir -p a/b/c
	@echo cqfd >a/cqfd_a.txt
	@echo cqfd_b >a/b/cqfd_b.txt
	@echo cqfd_c >a/b/c/cqfd_c.txt

clean:
	@rm -rf foo bar baz a/
