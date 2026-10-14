# Usage examples:
#   make deps
#   make check
#   make bootstrap                      # all workers (OS family picks the profile)
#   make bootstrap LIMIT=gpu-k8s-wk-1   # a single node
#   make bootstrap TAGS=gpu             # only the GPU role
ANSIBLE_PLAYBOOK ?= ansible-playbook
LIMIT ?= workers
TAGS  ?=

.PHONY: deps lint syntax check ping facts bootstrap operator verify site

deps:
	ansible-galaxy collection install -r requirements.yml

lint:
	yamllint .
	ansible-lint

syntax:
	$(ANSIBLE_PLAYBOOK) playbooks/site.yml --syntax-check

check: lint syntax

ping:
	ansible all -m ansible.builtin.ping

facts:
	ansible workers -m ansible.builtin.setup -a 'filter=ansible_os_family,ansible_distribution*'

bootstrap:
	$(ANSIBLE_PLAYBOOK) playbooks/bootstrap-workers.yml --limit $(LIMIT) $(if $(TAGS),--tags $(TAGS),)

operator:
	$(ANSIBLE_PLAYBOOK) playbooks/gpu-operator.yml

verify:
	$(ANSIBLE_PLAYBOOK) playbooks/verify-cluster.yml

site:
	$(ANSIBLE_PLAYBOOK) playbooks/site.yml
