# Ansible Role: `windows_patching`

Automates Windows Server patching via Ansible Automation Platform (AAP). This role optionally stops user-defined Windows services prior to patching, installs Windows Updates based on customizable update categories and KB blocklists, handles post-patch reboots, and generates a formatted summary report of all installed updates in the job output.

---

## Requirements & Prerequisites

### Collections
This role requires the **`ansible.windows`** collection. Ensure it is present in your AAP Execution Environment or defined in `collections/requirements.yml` at the root of your repository:

```yaml
---
collections:
  - name: ansible.windows
```

### Target Host Configuration (Windows Server 2019 / 2022)
* **Connection Method:** SSH (OpenSSH Server enabled on Windows) or WinRM.
* **Privileges:** The AAP Machine Credential must authenticate as a user in the local `Administrators` group.
* **Privilege Escalation (`become`):** Must be **disabled** (`become: false` / unchecked on the AAP Job Template). Using `sudo` against a PowerShell shell will cause the job to fail.

#### Recommended AAP Inventory Variables (SSH)
If connecting over SSH, configure the following variables on your Windows host or inventory group in AAP:

```yaml
ansible_connection: ssh
ansible_shell_type: powershell
ansible_ssh_args: "-o ControlMaster=no -o ControlPersist=no"
```

> **Note:** On the Windows target, ensure the OpenSSH `DefaultShell` registry property (`HKLM:\SOFTWARE\OpenSSH\DefaultShell`) is set to `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe` to prevent stdin pipe drops during large `win_updates` transfers.

---

## Role Variables

Default values are defined in `roles/windows_patching/defaults/main.yml`:

| Variable | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `custom_category_list` | `list` | `["CriticalUpdates", "SecurityUpdates"]` | Windows Update categories to search for and install. |
| `custom_reject_list` | `list` | `[]` | List of update titles, KB numbers (e.g., `KB5032198`), or Update IDs to exclude from installation. |
| `user_stopped_services` | `list` | `[]` | List of Windows service names (short names, e.g., `Spooler`) to stop before patching begins. |
| `perform_reboot` | `boolean` | `true` | Automatically reboot the Windows host if required by installed updates, then continue polling until complete. |

### Playbook Survey Normalization (`site.yml`)
When launched from AAP, `site.yml` accepts comma- or newline-separated strings from the Job Template survey and normalizes them into YAML lists in `pre_tasks`:
* `patch_categories_input` $\rightarrow$ `custom_category_list`
* `reject_list_input` $\rightarrow$ `custom_reject_list`
* `stopped_services_input` $\rightarrow$ `user_stopped_services`

---

## Repository Structure

```text
.
├── collections/
│   └── requirements.yml
├── roles/
│   └── windows_patching/
│       ├── defaults/
│       │   └── main.yml
│       └── tasks/
│           ├── main.yml
│           ├── patching.yml
│           └── services.yml
├── README.md
└── site.yml
```

---

## Setting Up the AAP Job Template & Survey

### 1. Job Template Settings
1. Navigate to **Resources > Templates** (or **Automation Execution > Templates**) in AAP and create or edit your **Job Template**.
2. Set **Playbook** to `site.yml`.
3. Attach your Windows **Inventory** and **Machine Credential**.
4. Ensure **Privilege Escalation** is **unchecked**.
5. Save the template.

### 2. Configure Survey Questions
Open the **Survey** tab on your Job Template, click **Add**, and create the following four survey questions:

#### Question 1: Update Categories
* **Prompt:** `Update Categories (comma or newline separated)`
* **Description:** `Windows Update categories to install (e.g., CriticalUpdates, SecurityUpdates, UpdateRollups, Updates, DefinitionUpdates).`
* **Answer Variable Name:** `patch_categories_input`
* **Answer Type:** `Textarea`
* **Required:** `Yes`
* **Default Answer:**
  ```text
  CriticalUpdates, SecurityUpdates
  ```

#### Question 2: Rejected Updates / KB Blocklist
* **Prompt:** `Rejected Updates / KB Blocklist (comma or newline separated)`
* **Description:** `Optional list of KB numbers or update titles to skip during patching (e.g., KB5032198).`
* **Answer Variable Name:** `reject_list_input`
* **Answer Type:** `Textarea`
* **Required:** `No`
* **Default Answer:** *(leave blank)*

#### Question 3: Services to Stop Before Patching
* **Prompt:** `Services to Stop Before Patching (comma or newline separated)`
* **Description:** `Optional list of Windows service short names to stop prior to running Windows Updates.`
* **Answer Variable Name:** `stopped_services_input`
* **Answer Type:** `Textarea`
* **Required:** `No`
* **Default Answer:** *(leave blank)*

#### Question 4: Reboot Behavior
* **Prompt:** `Reboot Server After Patching?`
* **Description:** `Select true to allow Ansible to automatically reboot the server if updates require a restart.`
* **Answer Variable Name:** `perform_reboot`
* **Answer Type:** `Multiple Choice (single select)`
* **Multiple Choice Options:**
  ```text
  true
  false
  ```
* **Required:** `Yes`
* **Default Answer:** `true`

### 3. Enable the Survey
After adding all four questions, toggle the **Survey Enabled** switch to **On** at the top of the Survey tab.

---

## Example Playbook (`site.yml`)

```yaml
---
- name: Apply Windows Server Patching
  hosts: all
  gather_facts: false

  pre_tasks:
    - name: Normalize Windows patching survey inputs from AAP
      ansible.builtin.set_fact:
        custom_category_list: >-
          {{
            (patch_categories_input | default('CriticalUpdates, SecurityUpdates') | string)
            .replace('\n', ',').split(',')
            | map('trim')
            | reject('equalto', '')
            | list
          }}
        custom_reject_list: >-
          {{
            (reject_list_input | default('') | string)
            .replace('\n', ',').split(',')
            | map('trim')
            | reject('equalto', '')
            | list
          }}
        user_stopped_services: >-
          {{
            (stopped_services_input | default('') | string)
            .replace('\n', ',').split(',')
            | map('trim')
            | reject('equalto', '')
            | list
          }}

  roles:
    - role: windows_patching
```

---

## Example Job Output

Upon completion, the `Update Report` task prints a summary block in the AAP job log:

```text
==================================
Host: win2k22-01
Number of Updates Installed: 4
System Rebooted: True
Updates Performed:
  - [INSTALLED] 2026-10 Cumulative Update for .NET Framework 3.5, 4.8 and 4.8.1 for Microsoft server operating system version 21H2 for x64 (KB5128928)
  - [INSTALLED] 2023-11 Cumulative Update for Microsoft server operating system version 21H2 for x64-based Systems (KB5032198)
  - [INSTALLED] Update for Microsoft Defender Antivirus antimalware platform - KB4052623 (Version 4.18.26100.4) - Current Channel (Broad)
  - [INSTALLED] 2022-02 Cumulative Update Preview for .NET Framework 3.5 and 4.8 for Microsoft server operating system version 21H2 for x64 (KB5010475)
==================================
```

---

## License

MIT-0