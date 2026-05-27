# Iac
all:
  children:
    cisco_switches:
      hosts:
        Sw1:
          ansible_host: 192.168.1.100

        Sw2:
          ansible_host: 192.168.1.101

      vars:
        ansible_connection: ansible.netcommon.network_cli
        ansible_network_os: cisco.ios.ios
        ansible_user: admin
        ansible_password: qwer
        ansible_become: true
        ansible_become_method: enable
        ansible_become_password: qwer

---
- name: Configure different Port-channel VLAN per switch
  hosts: cisco_switches
  gather_facts: false
  serial: 1

  vars:
    backup_dir: "./backup"
    run_id: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

    switch_config:
      Sw1:
        interface: "Port-channel1"
        vlan: 100
      Sw2:
        interface: "Port-channel3"
        vlan: 200

  tasks:
    - name: Create backup directory
      ansible.builtin.file:
        path: "{{ backup_dir }}"
        state: directory
      delegate_to: localhost
      run_once: true

    - name: Get running-config before change
      cisco.ios.ios_command:
        commands:
          - show running-config
      register: before_run

    - name: Save running-config before change
      ansible.builtin.copy:
        content: "{{ before_run.stdout[0] }}"
        dest: "{{ backup_dir }}/{{ inventory_hostname }}_before_{{ run_id }}.cfg"
      delegate_to: localhost

    - name: Configure target Port-channel
      cisco.ios.ios_config:
        parents:
          - "interface {{ switch_config[inventory_hostname].interface }}"
        lines:
          - switchport mode trunk
          - "switchport trunk allowed vlan add {{ switch_config[inventory_hostname].vlan }}"
        save_when: modified

    - name: Get running-config after change
      cisco.ios.ios_command:
        commands:
          - show running-config
      register: after_run

    - name: Save running-config after change
      ansible.builtin.copy:
        content: "{{ after_run.stdout[0] }}"
        dest: "{{ backup_dir }}/{{ inventory_hostname }}_after_{{ run_id }}.cfg"
      delegate_to: localhost

    - name: Compare before and after config
      ansible.builtin.shell: >
        diff -u
        {{ backup_dir }}/{{ inventory_hostname }}_before_{{ run_id }}.cfg
        {{ backup_dir }}/{{ inventory_hostname }}_after_{{ run_id }}.cfg
        > {{ backup_dir }}/{{ inventory_hostname }}_diff_{{ run_id }}.txt
      delegate_to: localhost
      failed_when: false
      changed_when: false
                
