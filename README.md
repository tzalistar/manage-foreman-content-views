# Managing Foreman Content Views with Ansible: A Markdown Guide

This guide provides clear, well-structured documentation in Markdown for the provided Ansible playbook. Each section explains a part of the playbook and includes code blocks for clarity.

---

## Table of Contents

- [Managing Foreman Content Views with Ansible: A Markdown Guide](#managing-foreman-content-views-with-ansible-a-markdown-guide)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Prerequisites](#prerequisites)
  - [Variables](#variables)
  - [Playbook Structure](#playbook-structure)
    - [1. Gather Content Views](#1-gather-content-views)
    - [2. Debug Target Environment](#2-debug-target-environment)
    - [3. Publish New Versions of CVs](#3-publish-new-versions-of-cvs)
    - [4. Wait for Background Tasks](#4-wait-for-background-tasks)
    - [5. Publish New Versions of CCVs](#5-publish-new-versions-of-ccvs)
    - [6. Promote CCVs to Target Environment](#6-promote-ccvs-to-target-environment)
    - [7. Clean Up Old Versions](#7-clean-up-old-versions)
  - [Tags and Selective Execution](#tags-and-selective-execution)
  - [Usage Examples](#usage-examples)
  - [Tips and Troubleshooting](#tips-and-troubleshooting)

---

## Overview

This playbook automates the management of [Foreman](https://theforeman.org/) Content Views (CVs) and Composite Content Views (CCVs):

- Publishes new versions of CVs and CCVs
- Promotes CCVs to a specified lifecycle environment
- Cleans up old versions, keeping only a set number of the latest

---

## Prerequisites

- **Ansible** installed
- **theforeman.foreman** collection installed:

```sh
ansible-galaxy collection install theforeman.foreman
```

- **Foreman server** accessible from the control node
- **API credentials** with permissions to manage Content Views

---

## Variables

You can override these variables via `--extra-vars` or in your inventory.

```yaml
vars:
  foreman_server_url: "{{ server_url | default('https://formanurl') }}"
  foreman_username: "{{ frm_username | default('someuser') }}"
  foreman_password: "{{ frm_password | default('somepass') }}"
  target_environment: "{{ lifecycle_environment | default('Enviroment') }}"
  organization: "{{ org | default('Default') }}"
  keep_versions: "{{ max_versions | default('2') }}"
```

| Variable | Default Value | Description |
| :-- | :-- | :-- |
| `server_url` | `https://formanurl` | Foreman server URL |
| `frm_username` | `someuser` | Foreman API username |
| `frm_password` | `somepass` | Foreman API password |
| `lifecycle_environment` | `Enviroment` | Target lifecycle environment for promotion |
| `org` | `Default` | Foreman organization name |
| `max_versions` | `2` | Number of versions to keep per Content View |

---

## Playbook Structure

### 1. Gather Content Views

Retrieves all Content Views in the specified organization.

```yaml
- name: Get all content views
  theforeman.foreman.content_view_info:
    server_url: "{{ foreman_server_url }}"
    username: "{{ foreman_username }}"
    password: "{{ foreman_password }}"
    organization: "{{ organization }}"
    validate_certs: false
  register: content_views
  tags:
    - publish
    - cleanup
    - promote-only
```

---

### 2. Debug Target Environment

Prints the target environment for confirmation.

```yaml
- name: Debug target target_environment
  ansible.builtin.debug:
    msg: "{{ target_environment }}"
  tags:
    - publish
    - cleanup
    - promote-only
```

---

### 3. Publish New Versions of CVs

Publishes a new version for each non-composite Content View (CV), skipping the "Default Organization View".

```yaml
- name: Publish a new version of each content view
  theforeman.foreman.content_view_version:
    server_url: "{{ foreman_server_url }}"
    username: "{{ foreman_username }}"
    password: "{{ foreman_password }}"
    organization: "{{ organization }}"
    validate_certs: false
    content_view: "{{ item.name }}"
  loop: "{{ content_views.content_views }}"
  when: not item.composite and 'Default Organization View' not in item.name
  ignore_errors: true
  tags:
    - publish
    - cv
  loop_control:
    pause: 300
```

---

### 4. Wait for Background Tasks

Waits 5 minutes to allow background publishing to complete.

```yaml
- name: Sleep for 300 seconds until all background stuff is done
  ansible.builtin.wait_for:
    timeout: 300
  delegate_to: localhost
  tags:
    - publish
    - promote
```

---

### 5. Publish New Versions of CCVs

Publishes a new version for each Composite Content View (CCV), skipping the "Default Organization View".

```yaml
- name: "Publish CCVs"
  theforeman.foreman.content_view_version:
    server_url: "{{ foreman_server_url }}"
    username: "{{ foreman_username }}"
    password: "{{ foreman_password }}"
    organization: "{{ organization }}"
    validate_certs: false
    content_view: "{{ item.name }}"
  loop: "{{ content_views.content_views }}"
  loop_control:
    pause: 300
  when: item.composite and 'Default Organization View' not in item.name
  ignore_errors: true
  tags:
    - publish
    - ccv
```

---

### 6. Promote CCVs to Target Environment

Ensures the latest version of each CCV is promoted to the target environment, if not already present.

```yaml
- name: "Ensure CCVs are on enviroment {{ target_environment }}"
  theforeman.foreman.content_view_version:
    server_url: "{{ foreman_server_url }}"
    username: "{{ foreman_username }}"
    password: "{{ foreman_password }}"
    organization: "{{ organization }}"
    validate_certs: false
    content_view: "{{ item.name }}"
    version: "{{ item.latest_version }}"
    lifecycle_environments: "{{ target_environment }}"
  loop: "{{ content_views.content_views }}"
  loop_control:
    pause: 300
  when: item.composite and 'Default Organization View' not in item.name and not
    (item.latest_version_environments | selectattr('name', 'equalto', target_environment) | list | length > 0)
  ignore_errors: true
  tags:
    - publish
    - promote-only
```

---

### 7. Clean Up Old Versions

Removes old versions of each CV and CCV, keeping only the latest `keep_versions`.

**For CCVs:**

```yaml
- name: "Clean up versions of each CCV. Versions to keep: {{ keep_versions }}"
  theforeman.foreman.content_view_version:
    server_url: "{{ foreman_server_url }}"
    username: "{{ foreman_username }}"
    password: "{{ foreman_password }}"
    organization: "{{ organization }}"
    validate_certs: false
    content_view: "{{ item.name }}"
    version: "{{ (item.latest_version | float - keep_versions | float) | string }}"
    state: absent
  when: (item.latest_version | float - keep_versions | float > keep_versions | float) and
        'Default Organization View' not in item.name and
        item.composite
  loop: "{{ content_views.content_views }}"
  loop_control:
    pause: 600
  ignore_errors: true
  tags:
    - cleanup
    - ccv
```

**For CVs:**

```yaml
- name: "Clean up versions of each CV. Versions to keep: {{ keep_versions }}"
  theforeman.foreman.content_view_version:
    server_url: "{{ foreman_server_url }}"
    username: "{{ foreman_username }}"
    password: "{{ foreman_password }}"
    organization: "{{ organization }}"
    validate_certs: false
    content_view: "{{ item.name }}"
    version: "{{ (item.latest_version | float - keep_versions | float) | string }}"
    state: absent
  when: (item.latest_version | float - keep_versions | float > keep_versions | float) and
        'Default Organization View' not in item.name and
        not item.composite
  loop: "{{ content_views.content_views }}"
  loop_control:
    pause: 600
  ignore_errors: true
  tags:
    - cleanup
    - cv
```

---

## Tags and Selective Execution

You can run specific parts of the playbook using tags:

- `publish`: Publish new versions of CVs and CCVs
- `promote-only`: Only promote CCVs to the environment
- `cleanup`: Remove old versions of CVs and CCVs
- `cv`: Tasks specific to Content Views
- `ccv`: Tasks specific to Composite Content Views

**Example:**

```sh
ansible-playbook playbook.yml --tags "publish"
```

---

## Usage Examples

**Run the full playbook:**

```sh
ansible-playbook playbook.yml
```

**Run only cleanup tasks:**

```sh
ansible-playbook playbook.yml --tags "cleanup"
```

**Override variables:**

```sh
ansible-playbook playbook.yml \
  -e "server_url=https://foreman.example.com" \
  -e "frm_username=admin" \
  -e "frm_password=secret" \
  -e "lifecycle_environment=Production" \
  -e "org=MyOrg" \
  -e "max_versions=3"
```

---

## Tips and Troubleshooting

- **Long Waits:** The playbook uses `wait_for` to allow background tasks to finish. Adjust the `timeout` if your environment is faster or slower.
- **Error Handling:** Tasks use `ignore_errors: true` to continue even if some content views fail. Review the output for any issues.
- **API Credentials:** Ensure your Foreman user has permissions to manage content views and versions.
- **Default Organization View:** The playbook skips the "Default Organization View" as it is typically managed by Foreman internally.
