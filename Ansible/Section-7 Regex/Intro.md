
# Regular Expressions (Regex) in Ansible

## 📌 Overview

A **Regular Expression (Regex)** is a sequence of characters that defines a search pattern. It is used to search, match, extract, replace, or validate text.

In Ansible, regex is commonly used to:

- Parse command output.
    
- Extract specific values.
    
- Validate strings.
    
- Replace text.
    
- Filter lists.
    
- Process network device configurations.
    

Regex is especially useful in **network automation**, where command outputs are plain text.

---

# Why Use Regex in Ansible?

Consider the following Cisco IOS output:

```text
R1# show ip interface brief

Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     192.168.1.1     YES manual up                    up
GigabitEthernet0/1     unassigned      YES unset  administratively down down
Loopback0              10.10.10.1      YES manual up                    up
```

Suppose you only want the IP addresses.

Instead of manually processing the text, regex can extract them automatically.

---

# Basic Regex Syntax

|Pattern|Meaning|Example Match|
|---|---|---|
|`.`|Any single character|`a`, `1`, `#`|
|`*`|Zero or more occurrences|`aaa`, empty string|
|`+`|One or more occurrences|`aaa`|
|`?`|Zero or one occurrence|Optional character|
|`^`|Start of line|`hostname` at the beginning|
|`$`|End of line|`up` at the end|
|`\d`|Digit|`1`, `5`, `9`|
|`\w`|Word character|`A`, `z`, `5`, `_`|
|`\s`|Whitespace|Space or tab|
|`[abc]`|Match one character|`a`, `b`, or `c`|
|`[^abc]`|Match anything except|Any character except `a`, `b`, or `c`|
|`[0-9]`|Digit between 0 and 9|`7`|
|`|`|OR operator|
|`( )`|Capture group|Stores matched text|

---

# Common Regex Examples

## Match a Hostname

Configuration:

```text
hostname CSR1
```

Regex:

```text
^hostname
```

Matches:

```text
hostname CSR1
```

---

## Match an IP Address

Regex:

```text
\d+\.\d+\.\d+\.\d+
```

Matches:

```text
192.168.1.1
10.10.10.10
172.16.5.254
```

---

## Match Interface Names

Regex:

```text
GigabitEthernet\d+/\d+
```

Matches:

```text
GigabitEthernet0/0
GigabitEthernet1/1
GigabitEthernet2/3
```

---

## Match VLAN Numbers

Regex:

```text
VLAN\s+\d+
```

Matches:

```text
VLAN 10
VLAN 100
```

---

# Regex Filters in Ansible

Ansible provides several Jinja2 filters for working with regex.

|Filter|Purpose|
|---|---|
|`regex_search`|Finds the first matching text.|
|`regex_findall`|Returns all matches.|
|`regex_replace`|Replaces matching text.|
|`regex_escape`|Escapes regex special characters.|

---

# `regex_search`

Searches for the first matching pattern.

Example:

```yaml
- name: Extract hostname
  ansible.builtin.debug:
    msg: "{{ 'hostname CSR1' | regex_search('hostname\\s+(\\S+)') }}"
```

Output:

```text
hostname CSR1
```

To extract only the hostname:

```yaml
- name: Extract hostname only
  ansible.builtin.debug:
    msg: "{{ 'hostname CSR1' | regex_search('hostname\\s+(\\S+)', '\\1') }}"
```

Output:

```text
CSR1
```

---

# `regex_findall`

Returns every match.

Example:

```yaml
- name: Extract IP addresses
  ansible.builtin.debug:
    msg: "{{ output | regex_findall('\\d+\\.\\d+\\.\\d+\\.\\d+') }}"
```

Input:

```text
192.168.1.1
10.10.10.1
172.16.1.1
```

Output:

```yaml
- 192.168.1.1
- 10.10.10.1
- 172.16.1.1
```

---

# `regex_replace`

Replace text using regex.

Example:

```yaml
- name: Replace hostname
  ansible.builtin.debug:
    msg: "{{ 'hostname R1' | regex_replace('R1','CSR1') }}"
```

Output:

```text
hostname CSR1
```

---

# Using Regex with Cisco IOS Output

Suppose this command is executed:

```yaml
- name: Show interfaces
  cisco.ios.ios_command:
    commands:
      - show ip interface brief
  register: interfaces
```

The output contains:

```text
GigabitEthernet0/0 192.168.1.1 YES manual up up
GigabitEthernet0/1 unassigned YES unset administratively down down
Loopback0 10.10.10.1 YES manual up up
```

Extract IP addresses:

```yaml
- name: Extract IPs
  ansible.builtin.debug:
    msg: "{{ interfaces.stdout[0] | regex_findall('\\d+\\.\\d+\\.\\d+\\.\\d+') }}"
```

Output:

```yaml
- 192.168.1.1
- 10.10.10.1
```

---

# Extract Interface Names

Example:

```yaml
- name: Extract interface names
  ansible.builtin.debug:
    msg: "{{ interfaces.stdout[0] | regex_findall('GigabitEthernet\\d+/\\d+') }}"
```

Output:

```yaml
- GigabitEthernet0/0
- GigabitEthernet0/1
```

---

# Validate an IPv4 Address

Regex:

```text
^\d+\.\d+\.\d+\.\d+$
```

Example:

```yaml
- name: Validate IP
  ansible.builtin.debug:
    msg: >
      {{ '192.168.1.10'
         is match('^\\d+\\.\\d+\\.\\d+\\.\\d+$') }}
```

Output:

```text
True
```

---

# Extract Hostname from Running Configuration

Configuration:

```text
hostname CSR1
service timestamps debug datetime
```

Playbook:

```yaml
- name: Extract hostname
  ansible.builtin.debug:
    msg: >
      {{
        running_config
        | regex_search('hostname\\s+(\\S+)', '\\1')
      }}
```

Output:

```text
CSR1
```

---

# Parse VLAN IDs

Configuration:

```text
vlan 10
vlan 20
vlan 100
```

Playbook:

```yaml
- name: Extract VLANs
  ansible.builtin.debug:
    msg: "{{ config | regex_findall('vlan\\s+(\\d+)') }}"
```

Output:

```yaml
- '10'
- '20'
- '100'
```

---

# Using Regex in `when` Conditions

Example:

```yaml
- name: Configure only Gigabit interfaces
  ansible.builtin.debug:
    msg: "Gigabit interface found"

  when: interface_name is match('GigabitEthernet.*')
```

If:

```yaml
interface_name: GigabitEthernet0/0
```

The task executes.

---

# Using Regex with `select`

Example:

```yaml
vars:
  interfaces:
    - GigabitEthernet0/0
    - FastEthernet0/0
    - Loopback0

tasks:
  - debug:
      msg: >
        {{
          interfaces
          | select('match','GigabitEthernet.*')
          | list
        }}
```

Output:

```yaml
- GigabitEthernet0/0
```

---

# Practical Network Automation Examples

## Find All Loopback Interfaces

```yaml
regex_findall('Loopback\\d+')
```

---

## Extract MAC Addresses

Regex:

```text
[0-9A-Fa-f]{4}\.[0-9A-Fa-f]{4}\.[0-9A-Fa-f]{4}
```

Matches:

```text
0011.2233.4455
AABB.CCDD.EEFF
```

---

## Extract Serial Numbers

Regex:

```text
SN:\\s+(\\S+)
```

---

## Find Interface Status

Regex:

```text
up|down
```

Matches:

```text
up
down
```

---

# Best Practices

- Keep regex patterns as simple as possible.
    
- Test patterns before using them in playbooks.
    
- Use raw strings or properly escape backslashes in YAML.
    
- Use capture groups (`( )`) to extract only the required values.
    
- Prefer resource modules (`ios_interfaces`, `ios_l3_interfaces`, etc.) over regex parsing when structured data is available. Use regex primarily for parsing CLI output from commands.
    

---

# Key Takeaways

|Regex Filter|Use|
|---|---|
|`regex_search`|Find the first matching value.|
|`regex_findall`|Return all matching values.|
|`regex_replace`|Replace matching text.|
|`match` test|Check if an entire string matches a pattern.|
|`search` test|Check if a pattern appears anywhere in a string.|

---

# Summary

Regular Expressions are a powerful tool for processing unstructured text in Ansible. They are widely used in network automation to parse Cisco IOS command outputs, validate values, extract information such as IP addresses, interface names, VLAN IDs, and hostnames, and transform text for further automation.

By combining regex with Ansible's Jinja2 filters, you can efficiently convert raw CLI output into structured data that can be used in conditions, loops, reports, and configuration validation.