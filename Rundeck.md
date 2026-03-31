---
title: Rundeck
description: Collection of Rundeck jobs and configs
published: true
date: 2026-03-31T07:23:26.758Z
tags: ansible, rundeck, openbao
editor: markdown
dateCreated: 2026-03-31T07:23:26.758Z
---

# Jobs
List of Jobs

## OpenBao
### Options

```yaml
- defaultTab: nodes
  description: |-
    DO CREATE SNAPSHOT OF VM svbao.example.com BEFORE ! ! !
    * Check Release notes before Upgrade: https://api.github.com/repos/openbao/openbao/releases/latest
    * Stops service, upgrades and starts service Openbao
  executionEnabled: false
  group: OpenBao
  id: 65d6f3b6-3c58-4223-8b53-a4fd79056937
  loglevel: INFO
  name: Upgrade Openbao
  nodeFilterEditable: false
  nodefilters:
    dispatch:
      excludePrecedence: true
      keepgoing: false
      rankOrder: ascending
      successOnEmptyNodeFilter: false
      threadcount: '1'
    filter: svbao.example.com
  nodesSelectedByDefault: true
  options:
  - description: 'Desired version number, or "auto" for latest'
    label: OpenBao Version
    name: ao_version
    regex: ^(\d+\.\d+\.\d+|\d+\.\d+)|auto$
    value: auto
  plugins:
    ExecutionLifecycle: {}
  scheduleEnabled: true
  sequence:
    commands:
    - configuration:
        ansible-become: 'true'
        ansible-encrypt-extra-vars: 'false'
        ansible-extra-vars: |-
          ---
          ao_version: ${option.ao_version}
          ...
        ansible-playbook-inline: "- hosts: svbao.example.com\n  gather_facts: true\n\
          \  vars:\n    ansible_become: true\n\n    # === Inputs ===\n    ao_version:\
          \ \"auto\"            # \"auto\" or e.g. \"2.3.2\" (with/without leading\
          \ 'v')\n    confirm_wait_seconds: 30      # set 0 to skip pause\n    arch:\
          \ \"amd64\"                 # adjust if needed; default fits x86_64\n\n\
          \    # === Internals ===\n    bao_addr: \"http://127.0.0.1:8201\"\n    bao_data_fallback:\
          \ \"/opt/openbao\"\n\n  environment:\n    BAO_ADDR: \"{{ bao_addr }}\"\n\
          \n  tasks:\n    - name: Ensure OpenBao GPG key is installed (for RPM verification)\n\
          \      ansible.builtin.rpm_key:\n        key: https://openbao.org/assets/openbao-gpg-pub-20240618.asc\n\
          \        state: present\n\n    # --- Resolve desired version (auto vs manual)\
          \ ---\n    - name: Get latest OpenBao release via GitHub API (auto mode)\n\
          \      ansible.builtin.uri:\n        url: https://api.github.com/repos/openbao/openbao/releases/latest\n\
          \        headers:\n          Accept: application/vnd.github+json\n     \
          \     User-Agent: ansible\n        return_content: true\n        status_code:\
          \ 200\n      register: gh_latest\n      when: ao_version | lower == \"auto\"\
          \n      failed_when: gh_latest.status != 200\n\n    - name: Set latest_version\
          \ + desired_version from API (auto)\n      ansible.builtin.set_fact:\n \
          \       latest_version: \"{{ (gh_latest.json.tag_name | default('v0.0.0'))\
          \ | regex_replace('^v','') }}\"\n        desired_version: \"{{ (gh_latest.json.tag_name\
          \ | default('v0.0.0')) | regex_replace('^v','') }}\"\n      when: ao_version\
          \ | lower == \"auto\"\n    \n    - name: Set desired_version from manual\
          \ input (strip leading 'v')\n      ansible.builtin.set_fact:\n        desired_version:\
          \ \"{{ (ao_version | string) | regex_replace('^v','') }}\"\n      when:\
          \ ao_version | lower != \"auto\"\n    \n    \n    - name: Guard – manual\
          \ desired version must not exceed GitHub latest\n      ansible.builtin.assert:\n\
          \        that:\n          - desired_version is version(latest_version, '<=')\n\
          \        fail_msg: >-\n          Requested version {{ desired_version }}\
          \ is higher than the latest\n          GitHub release {{ latest_version\
          \ }}. Aborting to prevent selecting\n          a non-existent release.\n\
          \      when: ao_version | lower != \"auto\"\n\n    # --- Current status\
          \ & upgrade decision ---\n    - name: Determine current status (JSON)\n\
          \      ansible.builtin.command: bao status -format=json\n      register:\
          \ status_before\n      changed_when: false\n      failed_when: false\n \
          \     \n    - name: Parse current status\n      ansible.builtin.set_fact:\n\
          \        bao_status_before: \"{{ (status_before.stdout | default('{}'))\
          \ | from_json }}\"\n      when: status_before.stdout is defined and status_before.stdout\
          \ != \"\"\n\n    - name: Preflight — current vs desired vs latest\n    \
          \  ansible.builtin.debug:\n        msg:\n          - \"Current version:\
          \ {{ bao_status_before.version | default('unknown') }}\"\n          - \"\
          Desired version: {{ desired_version }}\"\n          - \"GitHub latest :\
          \ {{ latest_version }}\"\n\n    - name: Compose RPM filename and URL\n \
          \     ansible.builtin.set_fact:\n        rpm_file: \"bao_{{ desired_version\
          \ }}_linux_{{ arch }}.rpm\"\n        rpm_url: \"https://github.com/openbao/openbao/releases/download/v{{\
          \ desired_version }}/bao_{{ desired_version }}_linux_{{ arch }}.rpm\"\n\n\
          \    - name: Show chosen version + URL (preflight)\n      ansible.builtin.debug:\n\
          \        msg:\n          - \"Chosen version: {{ desired_version }} (requested:\
          \ {{ ao_version }})\"\n          - \"RPM URL: {{ rpm_url }}\"\n\n    - name:\
          \ Wait for {{ confirm_wait_seconds }} seconds to allow cancellation if version\
          \ looks wrong\n      ansible.builtin.pause:\n        seconds: \"{{ confirm_wait_seconds\
          \ | int }}\"\n      when: (confirm_wait_seconds | int) > 0\n      \n   \
          \ # Compute version relationship with proper semantic comparison\n    -\
          \ name: Compute version relationship\n      ansible.builtin.set_fact:\n\
          \        is_newer_desired: \"{{ (bao_status_before.version | default('0.0.0'))\
          \ is version(desired_version, '<') }}\"\n        is_same_version: \"{{ (bao_status_before.version\
          \ | default('0.0.0')) is version(desired_version, '==') }}\"\n        is_older_desired:\
          \ \"{{ (bao_status_before.version | default('0.0.0')) is version(desired_version,\
          \ '>') }}\"\n        upgrade_needed: \"{{ (bao_status_before.version | default('0.0.0'))\
          \ is version(desired_version, '<') }}\"\n\n    - name: Guard – refuse downgrade\
          \ or no-op\n      ansible.builtin.assert:\n        that:\n          - is_newer_desired\n\
          \        fail_msg: >-\n          Requested version {{ desired_version }}\
          \ is not newer or equal than current\n          {{ bao_status_before.version\
          \ }}. Refusing downgrade/no-op.\n\n    - name: Log current vs desired\n\
          \      ansible.builtin.debug:\n        msg:\n          - \"Current version:\
          \ {{ bao_status_before.version | default('unknown') }}\"\n          - \"\
          Desired version: {{ desired_version }}\"\n          - \"Desired is newer:\
          \ {{ is_newer_desired }}\"\n          - \"Desired is same:  {{ is_same_version\
          \ }}\"\n          - \"Desired is older: {{ is_older_desired }}\"\n     \
          \     - \"Upgrade needed:  {{ upgrade_needed }}\"\n\n    # --- Safe stop\
          \ ---\n    - name: Stop service\n      ansible.builtin.service:\n      \
          \  name: openbao\n        state: stopped\n      when: upgrade_needed\n\n\
          \    # --- Fetch + install RPM (GPG verified by dnf because key is present)\
          \ ---\n    - name: Install/upgrade to {{ desired_version }}\n      ansible.builtin.dnf:\n\
          \        name: \"{{ rpm_url }}\"\n        state: present\n        disable_gpg_check:\
          \ false\n      when: upgrade_needed\n      register: dnf_out\n\n    - name:\
          \ Report any .rpmnew files to merge\n      ansible.builtin.shell: \"ls -1\
          \ /etc/openbao/*.rpmnew 2>/dev/null || true\"\n      register: rpmnew\n\
          \      changed_when: false\n      when: upgrade_needed\n\n    # --- Start\
          \ + verify ---\n    - name: Start service\n      ansible.builtin.service:\n\
          \        name: openbao\n        state: started\n      when: upgrade_needed\n\
          \n    - name: Wait for local listener to accept connections\n      ansible.builtin.wait_for:\n\
          \        host: 127.0.0.1\n        port: 8201\n        timeout: 20\n    \
          \  when: upgrade_needed\n\n    - name: Status after upgrade (JSON)\n   \
          \   ansible.builtin.command: bao status -format=json\n      register: status_after\n\
          \      changed_when: false\n      failed_when: false\n\n    - name: Parse\
          \ post-upgrade status\n      ansible.builtin.set_fact:\n        bao_status_after:\
          \ \"{{ (status_after.stdout | default('{}')) | from_json }}\"\n      when:\
          \ status_after.stdout is defined and status_after.stdout != \"\"\n\n   \
          \ - name: Assert target version is installed\n      when: upgrade_needed\n\
          \      ansible.builtin.assert:\n        that:\n          - bao_status_after.version\
          \ == desired_version\n        fail_msg: \"OpenBao version is {{ bao_status_after.version\
          \ }}, expected {{ desired_version }}.\"\n\n    - name: Post-upgrade notes\n\
          \      ansible.builtin.debug:\n        msg:\n          - \"Service is up.\
          \ Version: {{ bao_status_after.version }}. Sealed: {{ bao_status_after.sealed\
          \ }}.\"\n          - \"If sealed=true, unseal with your key shares (upgrade\
          \ tasks finish after unseal).\"\n          - >-\n            \"If present,\
          \ merge and restart after reviewing:\n            {{ (rpmnew.stdout_lines\
          \ | default([])) | join(', ') or 'none found' }}\"\n"
        ansible-ssh-passphrase-option: option.password
        ansible-ssh-use-agent: 'false'
      description: Upgrade Openbao
      nodeStep: true
      type: com.batix.rundeck.plugins.AnsiblePlaybookInlineWorkflowNodeStep
    keepgoing: false
    strategy: node-first
  user: example
  uuid: 28d6a3b6-8bc83-8945-8b53-a4fd19058954
```
Readable version of playbook
```yaml
- hosts: svbao.example.com
  gather_facts: true
  vars:
    ansible_become: true

    # === Inputs ===
    ao_version: "auto"            # "auto" or e.g. "2.3.2" (with/without leading 'v')
    confirm_wait_seconds: 30      # set 0 to skip pause
    arch: "amd64"                 # adjust if needed; default fits x86_64

    # === Internals ===
    bao_addr: "http://127.0.0.1:8201"
    bao_data_fallback: "/opt/openbao"

  environment:
    BAO_ADDR: "{{ bao_addr }}"

  tasks:
    - name: Ensure OpenBao GPG key is installed (for RPM verification)
      ansible.builtin.rpm_key:
        key: https://openbao.org/assets/openbao-gpg-pub-20240618.asc
        state: present

    # --- Resolve desired version (auto vs manual) ---
    - name: Get latest OpenBao release via GitHub API (auto mode)
      ansible.builtin.uri:
        url: https://api.github.com/repos/openbao/openbao/releases/latest
        headers:
          Accept: application/vnd.github+json
          User-Agent: ansible
        return_content: true
        status_code: 200
      register: gh_latest
      when: ao_version | lower == "auto"
      failed_when: gh_latest.status != 200

    - name: Set latest_version + desired_version from API (auto)
      ansible.builtin.set_fact:
        latest_version: "{{ (gh_latest.json.tag_name | default('v0.0.0')) | regex_replace('^v','') }}"
        desired_version: "{{ (gh_latest.json.tag_name | default('v0.0.0')) | regex_replace('^v','') }}"
      when: ao_version | lower == "auto"
    
    - name: Set desired_version from manual input (strip leading 'v')
      ansible.builtin.set_fact:
        desired_version: "{{ (ao_version | string) | regex_replace('^v','') }}"
      when: ao_version | lower != "auto"
    
    
    - name: Guard – manual desired version must not exceed GitHub latest
      ansible.builtin.assert:
        that:
          - desired_version is version(latest_version, '<=')
        fail_msg: >-
          Requested version {{ desired_version }} is higher than the latest
          GitHub release {{ latest_version }}. Aborting to prevent selecting
          a non-existent release.
      when: ao_version | lower != "auto"

    # --- Current status & upgrade decision ---
    - name: Determine current status (JSON)
      ansible.builtin.command: bao status -format=json
      register: status_before
      changed_when: false
      failed_when: false
      
    - name: Parse current status
      ansible.builtin.set_fact:
        bao_status_before: "{{ (status_before.stdout | default('{}')) | from_json }}"
      when: status_before.stdout is defined and status_before.stdout != ""

    - name: Preflight — current vs desired vs latest
      ansible.builtin.debug:
        msg:
          - "Current version: {{ bao_status_before.version | default('unknown') }}"
          - "Desired version: {{ desired_version }}"
          - "GitHub latest : {{ latest_version }}"

    - name: Compose RPM filename and URL
      ansible.builtin.set_fact:
        rpm_file: "bao_{{ desired_version }}_linux_{{ arch }}.rpm"
        rpm_url: "https://github.com/openbao/openbao/releases/download/v{{ desired_version }}/bao_{{ desired_version }}_linux_{{ arch }}.rpm"

    - name: Show chosen version + URL (preflight)
      ansible.builtin.debug:
        msg:
          - "Chosen version: {{ desired_version }} (requested: {{ ao_version }})"
          - "RPM URL: {{ rpm_url }}"

    - name: Wait for {{ confirm_wait_seconds }} seconds to allow cancellation if version looks wrong
      ansible.builtin.pause:
        seconds: "{{ confirm_wait_seconds | int }}"
      when: (confirm_wait_seconds | int) > 0
      
    # Compute version relationship with proper semantic comparison
    - name: Compute version relationship
      ansible.builtin.set_fact:
        is_newer_desired: "{{ (bao_status_before.version | default('0.0.0')) is version(desired_version, '<') }}"
        is_same_version: "{{ (bao_status_before.version | default('0.0.0')) is version(desired_version, '==') }}"
        is_older_desired: "{{ (bao_status_before.version | default('0.0.0')) is version(desired_version, '>') }}"
        upgrade_needed: "{{ (bao_status_before.version | default('0.0.0')) is version(desired_version, '<') }}"

    - name: Guard – refuse downgrade or no-op
      ansible.builtin.assert:
        that:
          - is_newer_desired
        fail_msg: >-
          Requested version {{ desired_version }} is not newer or equal than current
          {{ bao_status_before.version }}. Refusing downgrade/no-op.

    - name: Log current vs desired
      ansible.builtin.debug:
        msg:
          - "Current version: {{ bao_status_before.version | default('unknown') }}"
          - "Desired version: {{ desired_version }}"
          - "Desired is newer: {{ is_newer_desired }}"
          - "Desired is same:  {{ is_same_version }}"
          - "Desired is older: {{ is_older_desired }}"
          - "Upgrade needed:  {{ upgrade_needed }}"

    # --- Safe stop ---
    - name: Stop service
      ansible.builtin.service:
        name: openbao
        state: stopped
      when: upgrade_needed

    # --- Fetch + install RPM (GPG verified by dnf because key is present) ---
    - name: Install/upgrade to {{ desired_version }}
      ansible.builtin.dnf:
        name: "{{ rpm_url }}"
        state: present
        disable_gpg_check: false
      when: upgrade_needed
      register: dnf_out

    - name: Report any .rpmnew files to merge
      ansible.builtin.shell: "ls -1 /etc/openbao/*.rpmnew 2>/dev/null || true"
      register: rpmnew
      changed_when: false
      when: upgrade_needed

    # --- Start + verify ---
    - name: Start service
      ansible.builtin.service:
        name: openbao
        state: started
      when: upgrade_needed

    - name: Wait for local listener to accept connections
      ansible.builtin.wait_for:
        host: 127.0.0.1
        port: 8201
        timeout: 20
      when: upgrade_needed

    - name: Status after upgrade (JSON)
      ansible.builtin.command: bao status -format=json
      register: status_after
      changed_when: false
      failed_when: false

    - name: Parse post-upgrade status
      ansible.builtin.set_fact:
        bao_status_after: "{{ (status_after.stdout | default('{}')) | from_json }}"
      when: status_after.stdout is defined and status_after.stdout != ""

    - name: Assert target version is installed
      when: upgrade_needed
      ansible.builtin.assert:
        that:
          - bao_status_after.version == desired_version
        fail_msg: "OpenBao version is {{ bao_status_after.version }}, expected {{ desired_version }}."

    - name: Post-upgrade notes
      ansible.builtin.debug:
        msg:
          - "Service is up. Version: {{ bao_status_after.version }}. Sealed: {{ bao_status_after.sealed }}."
          - "If sealed=true, unseal with your key shares (upgrade tasks finish after unseal)."
          - >-
            "If present, merge and restart after reviewing:
            {{ (rpmnew.stdout_lines | default([])) | join(', ') or 'none found' }}"

```