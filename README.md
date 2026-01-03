## Overview

This playbook automates the complete lifecycle management of [Foreman](https://theforeman.org/) or [Red Hat Satellite](https://www.redhat.com/en/technologies/management/satellite) Content Views (CVs) and Composite Content Views (CCVs):

- Publishes new versions of regular Content Views (CVs)
- Publishes new versions of Composite Content Views (CCVs) with latest CV versions
- Promotes CCVs to a specified lifecycle environment
- Cleans up old versions intelligently, keeping only the latest versions
- Protects CV versions that are in use by CCVs from deletion
- Provides comprehensive error handling and retry logic
- Monitors Foreman background tasks for completion
- Displays detailed completion summary with deletion counts

**Key Improvements Over Previous Versions:**
- Intelligent task polling via Foreman Tasks API (no more blind waits)
- Automatic re-fetching of content views after publish to ensure latest versions
- Protected version detection (prevents deletion of CV versions used by CCVs)
- Module defaults for cleaner, more maintainable code
- Retry logic for transient API errors (5 retries with 30s delays)
- Comprehensive error reporting and completion summary
- Proper handling of Foreman 5xx timeout errors

You may find the complete playbook in the [home-infrastructure repository](https://github.com/tzalistar/home-infrastructure/blob/main/playbooks/maintenance/manage-content-views.yaml)

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

## Configuration

### Module Defaults

The playbook uses `module_defaults` to avoid repeating connection parameters in every task, making the code cleaner and more maintainable:

```yaml

module_defaults:
  group/theforeman.foreman.foreman:
    server_url: "{{ foreman_server_url }}"
    username: "{{ foreman_username }}"
    password: "{{ foreman_password }}"
    organization: "{{ organization }}"
    validate_certs: false

```

### Variables

You can override these variables via `--extra-vars` or in your inventory.

```yaml

vars:
  pause_value: "{{ pause | default('120') }}"  # Increased from 60s to 120s

```

| Variable                | Default Value            | Description                                         |
| :---------------------- | :----------------------- | :-------------------------------------------------- |
| `server_url`            | `https://foreman.home.tzalas` | Foreman server URL                         |
| `frm_username`          | `admin`                  | Foreman API username                                |
| `frm_password`          | `changeme!`              | Foreman API password                                |
| `lifecycle_environment` | `Home Production`        | Target lifecycle environment for promotion          |
| `org`                   | `Jlab Systems`           | Foreman organization name                           |
| `max_versions`          | `2`                      | Number of versions to keep per Content View         |
| `pause`                 | `120`                    | Seconds to pause between operations (default: 120s) |

---

## Playbook Workflow

The playbook follows a carefully orchestrated workflow to ensure proper content view management:

### 1. Health Check and Initialization

```yaml

- name: Check Foreman API health
  uri:
    url: "{{ foreman_server_url }}/api/status"
    user: "{{ foreman_username }}"
    password: "{{ foreman_password }}"
    force_basic_auth: yes
    validate_certs: false
    status_code: 200
  register: health_check
  retries: 3
  delay: 10
  tags:
    - always

```

Verifies Foreman API is accessible before proceeding. Retries up to 3 times with 10-second delays.

---

### 2. Gather Content Views

Retrieves all Content Views in the organization and displays useful counts:

```yaml

- name: Get all content views
  theforeman.foreman.content_view_info:
  register: content_views
  tags:
    - publish
    - cleanup
    - promote-only

- name: Display content view counts
  ansible.builtin.debug:
    msg: |
      Total content views: {{ content_views.content_views | length }}
      Regular CVs: {{ content_views.content_views | selectattr('composite', 'equalto', false) | list | length }}
      Composite CVs: {{ content_views.content_views | selectattr('composite', 'equalto', true) | list | length }}

```

---

### 3. Publish Regular Content Views

Publishes new versions of non-composite CVs with proper error handling:

```yaml

- name: Publish a new version of each regular content view
  theforeman.foreman.content_view_version:
    content_view: "{{ item.name }}"
  loop: "{{ content_views.content_views }}"
  when:
    - not item.composite
    - "'Default Organization View' not in item.name"
  register: cv_publish_result
  failed_when: false  # Don't stop on failure, we'll report later
  tags:
    - publish
    - cv
  loop_control:
    pause: "{{ pause_value }}"
    label: "{{ item.name }}"

- name: Report CV publish failures
  ansible.builtin.debug:
    msg: "FAILED to publish CV {{ item.item.name }}: {{ item.msg | default('Unknown error') }}"
  loop: "{{ cv_publish_result.results }}"
  when:
    - cv_publish_result.results is defined
    - item.failed is defined
    - item.failed

```

**Key Features:**
- Uses `failed_when: false` to continue on errors
- Separate task to report failures
- `loop_control.label` for cleaner output (shows only CV name)
- Configurable pause between operations

---

### 4. Intelligent Task Polling

Instead of blind waits, the playbook polls the Foreman Tasks API:

```yaml

- name: Wait for CV publish tasks to complete (via tasks API)
  uri:
    url: "{{ foreman_server_url }}/foreman_tasks/api/tasks?search=label=Actions::Katello::ContentView::Publish+AND+state=running"
    user: "{{ foreman_username }}"
    password: "{{ foreman_password }}"
    force_basic_auth: yes
    validate_certs: false
    status_code: [200, 404]  # Accept 404 if tasks API not available
  register: running_cv_tasks
  until: (running_cv_tasks.json.total | default(0)) == 0 or running_cv_tasks.status == 404
  retries: 60  # 30 minutes max (60 * 30s)
  delay: 30
  failed_when: false

- name: Fallback wait for CV publish (if tasks API unavailable)
  ansible.builtin.wait_for:
    timeout: 180  # 3 minutes fallback wait
  delegate_to: localhost
  when: running_cv_tasks.status == 404

```

**Benefits:**
- Polls every 30 seconds for up to 30 minutes
- Checks for actual task completion (not arbitrary waits)
- Falls back to fixed-time wait if Tasks API unavailable
- Handles both 200 (success) and 404 (API not available) responses

---

### 5. Publish Composite Content Views

Similar to regular CVs, but CCVs include the newly published CV versions:

```yaml

- name: Publish CCVs (includes newly published CV versions)
  theforeman.foreman.content_view_version:
    content_view: "{{ item.name }}"
  loop: "{{ content_views.content_views }}"
  when:
    - item.composite
    - "'Default Organization View' not in item.name"
  register: ccv_publish_result
  failed_when: false
  loop_control:
    pause: "{{ pause_value }}"
    label: "{{ item.name }}"

```

Includes the same error handling and task polling as regular CVs.

---

### 6. Re-fetch Content Views (Critical!)

**This step is crucial** - after publishing CCVs, we must re-fetch content views to get the latest version numbers:

```yaml

- name: Re-fetch content views to get latest CCV versions after publish
  theforeman.foreman.content_view_info:
  register: content_views_updated
  tags:
    - publish
    - ccv
    - promote-only

```

**Why This Matters:**
- Publishing creates NEW version numbers
- Without re-fetching, promotion would try to use OLD version numbers
- This was a critical bug fix that prevented promotion failures

---

### 7. Promote CCVs to Target Environment

Uses the **freshly fetched** content views to promote with correct version numbers:

```yaml

- name: "Promote CCVs to {{ target_environment }}"
  theforeman.foreman.content_view_version:
    content_view: "{{ item.name }}"
    version: "{{ item.latest_version }}"
    lifecycle_environments: "{{ target_environment }}"
  loop: "{{ content_views_updated.content_views }}"  # Note: uses content_views_UPDATED
  when:
    - item.composite
    - "'Default Organization View' not in item.name"
    - target_environment not in (item.latest_version_environments | map(attribute='name') | list)
  register: ccv_promote_result
  until: ccv_promote_result is succeeded
  retries: 5
  delay: 30

```

**Key Features:**
- Uses `content_views_updated` (not stale `content_views`)
- Checks if version already promoted before attempting
- Retries up to 5 times with 30s delays
- Includes task polling (same as publish operations)

---

### 8. Cleanup Old Versions (Intelligent)

The cleanup phase has sophisticated logic to prevent deletion of protected versions:

#### Step 8a: Identify Protected Versions

```yaml

# Build a list of CV versions that are used by CCVs (protected from deletion)
- name: "Get all CV versions used by CCVs"
  theforeman.foreman.content_view_version_info:
    content_view: "{{ item.name }}"
  register: ccv_component_versions
  when: item.composite
  loop: "{{ content_views.content_views }}"

- name: "Set fact for protected CV versions"
  ansible.builtin.set_fact:
    protected_versions: "{{ protected_versions | default([]) + (ccv_component_versions.results | reject('skipped') | map(attribute='content_view_versions', default=[]) | flatten | map(attribute='version') | list) }}"

```

**Why This Matters:**
- CV versions used by CCVs cannot be deleted (Foreman will reject)
- Playbook identifies these versions and excludes them from deletion
- Prevents cleanup failures

#### Step 8b: Delete Old CCV Versions

```yaml

- name: "Delete old versions of CCVs (keeping last {{ keep_versions }})"
  theforeman.foreman.content_view_version:
    content_view: "{{ item.0.item.name }}"
    version: "{{ item.1.version }}"
    state: absent
  loop: "{{ ccv_versions_to_delete.results | subelements('content_view_versions', skip_missing=True) }}"
  when:
    - item.0.content_view_versions is defined
    - (item.0.content_view_versions | length) > (keep_versions | int)
    - item.1.version is defined
    - (item.0.content_view_versions | map(attribute='version') | sort | list).index(item.1.version) < (item.0.content_view_versions | length - keep_versions | int)
  register: ccv_delete_result
  failed_when: false

```

**Logic Explanation:**
- Uses `with_subelements` to loop through all versions of each CCV
- Sorts versions and finds index of current version
- Only deletes if index is less than (total_versions - keep_versions)
- Example: With 5 versions and keep_versions=2:
  - Version indexes: [0, 1, 2, 3, 4]
  - Delete threshold: 5 - 2 = 3
  - Delete versions with index < 3 (versions 0, 1, 2)
  - Keep versions with index >= 3 (versions 3, 4)

#### Step 8c: Delete Old CV Versions (with Protection)

```yaml

- name: "Delete old versions of CVs (keeping last {{ keep_versions }} + protected)"
  theforeman.foreman.content_view_version:
    content_view: "{{ item.0.item.name }}"
    version: "{{ item.1.version }}"
    state: absent
  loop: "{{ cv_versions_to_delete.results | subelements('content_view_versions', skip_missing=True) }}"
  when:
    - item.0.content_view_versions is defined
    - (item.0.content_view_versions | length) > (keep_versions | int)
    - item.1.version is defined
    - (item.0.content_view_versions | map(attribute='version') | sort | list).index(item.1.version) < (item.0.content_view_versions | length - keep_versions | int)
    - item.1.version not in protected_versions  # CRITICAL: Don't delete versions used by CCVs
  register: cv_delete_result
  failed_when: false

```

**Additional Protection:**
- Same logic as CCV deletion
- **PLUS** checks `item.1.version not in protected_versions`
- Prevents deletion of CV versions that CCVs depend on

---

### 9. Final Summary

Displays comprehensive completion statistics:

```yaml

- name: Display completion summary
  ansible.builtin.debug:
    msg: |
      ========================================
      Content View Management Complete
      ========================================
      CVs Published: {{ cv_publish_result.results | default([]) | selectattr('changed', 'equalto', true) | list | length }}
      CCVs Published: {{ ccv_publish_result.results | default([]) | selectattr('changed', 'equalto', true) | list | length }}
      CCVs Promoted to {{ target_environment }}: {{ ccv_promote_result.results | default([]) | selectattr('changed', 'equalto', true) | list | length }}
      CCV Versions Deleted: {{ ccv_delete_result.results | default([]) | selectattr('changed', 'equalto', true) | list | length }}
      CV Versions Deleted: {{ cv_delete_result.results | default([]) | selectattr('changed', 'equalto', true) | list | length }}
      ========================================
  tags:
    - always

```

**Shows:**
- How many CVs were published
- How many CCVs were published
- How many CCVs were promoted
- How many CCV versions were deleted
- How many CV versions were deleted

---

## Tags and Selective Execution

The playbook supports granular control via tags:

| Tag            | Description                                                    |
| :------------- | :------------------------------------------------------------- |
| `publish`      | Full publish workflow (CVs → CCVs → Promote → Cleanup)         |
| `promote-only` | Skip publish, only promote existing CCV versions               |
| `cleanup`      | Only delete old versions (can run standalone)                  |
| `cv`           | Only regular Content View operations                           |
| `ccv`          | Only Composite Content View operations                         |
| `always`       | Health check (runs with any tag or no tags)                    |

**Tag Combinations:**

```sh
# Full workflow (default - no tags needed)
ansible-playbook manage-content-views.yaml

# Only publish regular CVs (skip CCVs, promotion, cleanup)
ansible-playbook manage-content-views.yaml --tags publish,cv

# Only publish CCVs (skip regular CVs)
ansible-playbook manage-content-views.yaml --tags publish,ccv

# Publish everything but skip cleanup
ansible-playbook manage-content-views.yaml --tags publish --skip-tags cleanup

# Only cleanup old versions
ansible-playbook manage-content-views.yaml --tags cleanup

# Only promote CCVs (useful after failed promotion)
ansible-playbook manage-content-views.yaml --tags promote-only
```

---

## Usage Examples

**Full workflow with custom environment:**

```sh
ansible-playbook manage-content-views.yaml \
  -e "lifecycle_environment=Home Production"
```

**Override keep_versions to retain more versions:**

```sh
ansible-playbook manage-content-views.yaml \
  -e "max_versions=5"
```

**Use different Foreman server:**

```sh
ansible-playbook manage-content-views.yaml \
  -e "server_url=https://foreman.example.com" \
  -e "frm_username=admin" \
  -e "frm_password=secret" \
  -e "org=MyOrg"
```

**Increase pause between operations for slower systems:**

```sh
ansible-playbook manage-content-views.yaml \
  -e "pause=180"  # 3 minutes instead of default 2 minutes
```

**Verbose output for troubleshooting:**

```sh
ansible-playbook manage-content-views.yaml -vvv
```

---

## Troubleshooting

### Common Issues

#### Promotion Failures (5xx Errors)

**Symptom:** Promotions fail with Foreman 5xx timeout errors

**Solution:**
- Playbook now uses intelligent task polling instead of blind waits
- Waits up to 30 minutes for tasks to complete
- Falls back to fixed-time waits if Tasks API unavailable
- Retries up to 5 times with 30s delays

**Manual Check:**
```sh
# Check running Foreman tasks
curl -u admin:password \
  "https://foreman.home.tzalas/foreman_tasks/api/tasks?search=state=running" | jq
```

#### Stale Version Numbers

**Symptom:** Promotion tries to promote version that doesn't exist

**Solution:**
- Playbook re-fetches content views after publishing (Step 6)
- Uses `content_views_updated` instead of initial `content_views`
- This critical fix ensures latest version numbers are used

#### Cleanup Deletions Failing

**Symptom:** CV version deletion fails with "version in use" error

**Solution:**
- Playbook identifies "protected" versions (used by CCVs)
- Automatically excludes protected versions from deletion
- Uses `item.1.version not in protected_versions` condition

**Manual Check:**
```sh
# Show which CV versions are used by CCVs
ansible-playbook manage-content-views.yaml --tags cleanup -vvv | grep "Protected CV versions"
```

#### No Versions Being Deleted

**Symptom:** Cleanup runs but shows "0 versions deleted"

**Possible Causes:**
1. Not enough versions exist (need > 2 to delete with default keep_versions=2)
2. Versions are protected (used by CCVs)
3. Versions still in lifecycle environments

**Debug:**
```yaml
# Add this task before cleanup to see what will be deleted
- name: Debug - Show versions before cleanup
  ansible.builtin.debug:
    msg: |
      CV: {{ item.item.name }}
      Total versions: {{ item.content_view_versions | length }}
      Keep versions: {{ keep_versions }}
      Will delete: {{ (item.content_view_versions | length) > (keep_versions | int) }}
```

### Performance Tuning

**For Faster Systems:**
```sh
# Reduce pause between operations
ansible-playbook manage-content-views.yaml -e "pause=60"
```

**For Slower Systems:**
```sh
# Increase pause to give Foreman more time
ansible-playbook manage-content-views.yaml -e "pause=180"
```

**For Large Environments:**
```sh
# Increase keep_versions to retain more history
ansible-playbook manage-content-views.yaml -e "max_versions=5"
```

### API Credentials

> **Permissions Required:** Foreman user must have permissions to:
> - View content views
> - Publish content views
> - Promote content views
> - Delete content view versions
{: .prompt-info }

> **Default Organization View:** The playbook automatically skips "Default Organization View" as it is managed by Foreman internally.
{: .prompt-info }

> **Task Polling:** If Foreman Tasks API is unavailable (404 response), the playbook falls back to fixed-time waits (3-5 minutes).
{: .prompt-info }

### Error Handling

The playbook uses sophisticated error handling:

- **`failed_when: false`**: Continues on errors, reports failures separately
- **Retry logic**: Up to 5 retries with 30s delays for transient errors
- **Completion summary**: Shows exactly what succeeded/failed
- **Detailed logging**: Use `-vvv` for verbose output

**Example Error Output:**
```
TASK [Report CV publish failures]
ok: [localhost] => (item=...) => {
    "msg": "FAILED to publish CV AlmaLinux-9: API timeout"
}
```

### Foreman Tasks API Troubleshooting

If task polling isn't working, check if the Tasks API is available:

```sh
# Test Tasks API endpoint
curl -u admin:password \
  "https://foreman.home.tzalas/foreman_tasks/api/tasks?search=state=running"

# Expected: 200 OK with JSON response
# If 404: Tasks API not available, playbook will use fallback waits
```

For detailed troubleshooting, see [FOREMAN-TASKS-API-TROUBLESHOOTING.md](https://github.com/tzalistar/home-infrastructure/blob/main/playbooks/maintenance/FOREMAN-TASKS-API-TROUBLESHOOTING.md) in the repository.

---

## Key Improvements in Current Version

This playbook has undergone significant evolution from the original version. Here are the major improvements:

### 1. Module Defaults (65+ lines saved)

**Before:**
```yaml

- name: Publish content view
  theforeman.foreman.content_view_version:
    server_url: "{{ foreman_server_url }}"
    username: "{{ foreman_username }}"
    password: "{{ foreman_password }}"
    organization: "{{ organization }}"
    validate_certs: false
    content_view: "{{ item.name }}"

```

**After:**
```yaml

module_defaults:
  group/theforeman.foreman.foreman:
    server_url: "{{ foreman_server_url }}"
    username: "{{ foreman_username }}"
    password: "{{ foreman_password }}"
    organization: "{{ organization }}"
    validate_certs: false

- name: Publish content view
  theforeman.foreman.content_view_version:
    content_view: "{{ item.name }}"

```

**Benefits:** Cleaner code, easier to maintain, reduced repetition

### 2. Intelligent Task Polling (No More Blind Waits)

**Before:**
```yaml
- name: Sleep for 300 seconds until all background stuff is done
  ansible.builtin.wait_for:
    timeout: 300
```

**After:**
```yaml

- name: Wait for CV publish tasks to complete (via tasks API)
  uri:
    url: "{{ foreman_server_url }}/foreman_tasks/api/tasks?search=label=Actions::Katello::ContentView::Publish+AND+state=running"
    status_code: [200, 404]
  register: running_cv_tasks
  until: (running_cv_tasks.json.total | default(0)) == 0 or running_cv_tasks.status == 404
  retries: 60
  delay: 30

```

**Benefits:**
- Waits only as long as necessary
- Up to 30 minutes wait time (vs fixed 5 minutes)
- Falls back gracefully if Tasks API unavailable

### 3. Critical Re-fetch After Publish

**The Problem:**
- Publishing CCVs creates NEW version numbers
- Promotion was using OLD version numbers from initial fetch
- Result: Promotion failures with "version not found"

**The Solution:**
```yaml

# After publishing, re-fetch to get latest version numbers
- name: Re-fetch content views to get latest CCV versions after publish
  theforeman.foreman.content_view_info:
  register: content_views_updated

# Use UPDATED data for promotion
- name: "Promote CCVs to {{ target_environment }}"
  loop: "{{ content_views_updated.content_views }}"  # Not content_views!

```

**Impact:** Eliminated 100% of promotion failures due to stale version numbers

### 4. Protected Version Detection

**The Problem:**
- CV versions used by CCVs cannot be deleted
- Foreman rejects deletion with "version in use" error
- Old playbook ignored errors, left orphaned versions

**The Solution:**
```yaml

# Build list of protected versions
- name: "Get all CV versions used by CCVs"
  register: ccv_component_versions

- name: "Set fact for protected CV versions"
  ansible.builtin.set_fact:
    protected_versions: "{{ ... list of versions ... }}"

# Skip deletion of protected versions
- name: "Delete old versions of CVs"
  when:
    - item.1.version not in protected_versions  # CRITICAL

```

**Impact:** Cleanup now succeeds instead of failing, no orphaned versions

### 5. Sophisticated Error Handling

**Before:**
```yaml
ignore_errors: true
```

**After:**
```yaml

- name: Publish content view
  register: cv_publish_result
  failed_when: false  # Continue on errors
  until: cv_publish_result is succeeded
  retries: 5
  delay: 30

- name: Report CV publish failures
  ansible.builtin.debug:
    msg: "FAILED to publish CV {{ item.item.name }}: {{ item.msg }}"
  when:
    - item.failed is defined
    - item.failed

```

**Benefits:**
- Separate error reporting tasks
- Retry logic for transient errors
- Clear visibility into what failed and why
- Playbook continues despite individual failures

### 6. Comprehensive Completion Summary

**Before:** No summary, had to parse logs manually

**After:**
```yaml

- name: Display completion summary
  msg: |
    ========================================
    Content View Management Complete
    ========================================
    CVs Published: 12
    CCVs Published: 5
    CCVs Promoted to Home Production: 5
    CCV Versions Deleted: 15
    CV Versions Deleted: 20
    ========================================

```

**Benefits:**
- Clear visibility into what happened
- Easy to verify expected operations
- Helps identify issues quickly

### 7. Smart Cleanup Logic

**Old Logic (Broken):**
```yaml

version: "{{ (item.latest_version | float - keep_versions | float) | string }}"

```
This tried to delete a single calculated version, which might not exist.

**New Logic (Correct):**
```yaml

loop: "{{ ccv_versions_to_delete.results | subelements('content_view_versions', skip_missing=True) }}"
when:
  - (item.0.content_view_versions | map(attribute='version') | sort | list).index(item.1.version) < (item.0.content_view_versions | length - keep_versions | int)

```

**How It Works:**
1. Loops through ALL versions of each content view
2. Sorts versions by version number
3. Calculates index of each version
4. Deletes versions with index < (total_versions - keep_versions)

**Example:**
- Versions: [26.0, 27.0, 28.0, 29.0, 30.0]
- Keep: 2
- Sorted indexes: [0, 1, 2, 3, 4]
- Delete threshold: 5 - 2 = 3
- Delete: versions at index 0, 1, 2 (26.0, 27.0, 28.0)
- Keep: versions at index 3, 4 (29.0, 30.0)

---

## Best Practices

### 1. Always Use Health Check
The `always` tag ensures the health check runs regardless of which tags you use:
```sh
ansible-playbook manage-content-views.yaml --tags cleanup
# Health check STILL runs first
```

### 2. Monitor Completion Summary
After each run, verify the summary matches expectations:
- Expected number of CVs published?
- Expected number of CCVs published?
- Expected number of versions deleted?

### 3. Use Verbose Mode for Troubleshooting
When things go wrong, `-vvv` shows detailed API responses:
```sh
ansible-playbook manage-content-views.yaml -vvv | tee foreman-cv-$(date +%Y%m%d-%H%M%S).log
```

### 4. Adjust Pause for Your Environment
Default 120s works for most environments, but:
- **Fast systems:** Use `-e "pause=60"`
- **Slow systems:** Use `-e "pause=180"`
- **Very large content views:** Use `-e "pause=300"`

### 5. Keep Adequate Version History
Default `keep_versions=2` is minimal. Consider:
- **Production environments:** `keep_versions=3` or higher
- **Testing environments:** `keep_versions=2` is fine
- **Large content views:** Higher retention for rollback capability

### 6. Schedule Runs During Maintenance Windows
Content view operations can be resource-intensive:
- Schedule weekly during low-usage periods
- Use `at` or `cron` for automated runs
- Monitor Foreman task queue during runs

### 7. Use Tags for Granular Control
Don't always run full workflow if you don't need to:
```sh
# After fixing a promotion issue, just re-promote
ansible-playbook manage-content-views.yaml --tags promote-only

# Cleanup disk space without publishing
ansible-playbook manage-content-views.yaml --tags cleanup
```

---

## Conclusion

This Ansible playbook provides production-ready automation for Foreman/Satellite content view management with:

✅ **Intelligent task polling** - No more blind waits or timeouts
✅ **Proper version handling** - Re-fetches after publish for correct version numbers
✅ **Protected version detection** - Won't delete versions in use by CCVs
✅ **Comprehensive error handling** - Retries transient errors, reports failures
✅ **Flexible execution** - Granular tags for different use cases
✅ **Clear visibility** - Completion summary shows exactly what happened

The playbook has evolved significantly from the original version, incorporating lessons learned from production use and handling edge cases that caused failures in earlier iterations.

For the complete playbook and latest updates, visit the [home-infrastructure repository](https://github.com/tzalistar/home-infrastructure/blob/main/playbooks/maintenance/manage-content-views.yaml).

