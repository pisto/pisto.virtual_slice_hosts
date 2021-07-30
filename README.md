Ansible role `pisto.virtual_slice_hosts`
=========

This utility role uses the `add_host` (`ansible.builtin.add_host`) module to create virtual "slices" of your inventory
hosts, in order to run tasks in parallel on each virtual slice.

You can parallelize loops very easily: consider this playbook, where the role `run-job` is executed to run some work,
out of a list of work items `work_items`:
```yaml
- name: Use run-jobs role in a loop
  hosts: all
  tasks:
  - name: Launch all job items
    loop: "{{ work_items }}"
    include_role:
      name: run-job
    vars:
      id: "{{ item }}"
```

The above playbook would run hosts in parallel (according to the number of forks), but it would run each element in
`work_items` serially. It is also not possible to use `async` and `poll: 0`, because `include_role` does not support
async tasks.

With `pisto.virtual_slice_hosts`, you can create "virtual" copies of your hosts that are added in the in-memory inventory of
Ansible in a `virtual` group. The new hosts will hold slices of the whole `work_items` array, and so Ansible can
parallelize operations at the host level!

First install it:
```bash
ansible-galaxy install pisto.virtual_slice_hosts
```

The serial playbook can be parallelized as below:
```yaml
# Prepend this play to your playbook:
- name: Prepare work items and create virtual slice hosts
  hosts: all
  tasks:
  - name: Create virtual slice hosts
    include_role:
      name: pisto.virtual_slice_hosts
    vars:
      payload_var: work_items     # the name of the host variable that holds the list of work items
      batch: 10                   # the maximum batch size

# Minimal modifications to the original play:
- name: Use run-jobs role in a loop for each virtual slice host
  hosts: all,&virtual             # notice the modified host pattern that selects the newly added hosts
  strategy: free                  # this is important for performance! Allows hosts to run independently of each other
  tasks:
  - name: Launch all job items
    loop: "{{ work_items }}"      # `work_items` here will be just a slice (max size 10) of the original list
    include_role:
      name: run-job
    vars:
      id: "{{ item }}"
```

For example, if you have `some-host` with a `work_items` list of size 100, the above playbook will run the `run-job`
role for 10 different virtual hosts `some-host.slice-[0-9]`, and each of these hosts will have a slice of size 10 of the
original `work_items` list. The virtual hosts will run the tasks in parallel, as much as the number of [forks](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-forks) setting allows.

Creation of the virtual host
--------------

The payload variable is selected with the `payload_var`. The variable is expected to hold a list, which will be split
in slices of size equal to the `batch` parameter. The resulting number of slices determines the number of virtual hosts
to create. The virtual hosts will see their own slice under the variable referenced by `payload_var`.

The virtual hosts inherit the same groups of the real hosts. Additionally, they are placed in an additional group, by
default `virtual`: this group can be used to select the virtual hosts in host patterns. This group name is controlled
by the variable `group_name`.

This role uses internally the [`add_host` module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/add_host_module.html) It is not possible to remove
the new hosts. It is however possible to use this role multiple times, with a different value for the the `group_name`
parameter to select between different parallel sections of a playbook.

The virtual hosts do not inherit automatically all the real host variables. Several connection, user and privilege
escalation-related variables are copied over by default. If you need additional variables to be copied over, list them
in the list `copy_vars`.

Role Variables
--------------

| variable             | description                                                                                                            |
|----------------------|------------------------------------------------------------------------------------------------------------------------|
| `host_suffix`        | suffix of the generated host name, default `slice`                                                                     |
| `group_name`         | group name of the new virtual hosts (default `virtual`)                                                                |
| `payload_var`        | name of the variable that holds the "work" list, that will be split in equal batches across virtual hosts              |
| `batch`              | size of the batch                                                                                                      |
| `copy_vars`          | list of copied variables (default `[]`)                                                                                |
| `implicit_copy_vars` | additional list of copied variables [connection, user and privilege escalation-related variables](https://github.com/pisto/pisto.virtual_slice_hosts/blob/main/defaults/main.yml#L4-L14) |

License
-------

MIT

Author Information
------------------

Lorenzo Pistone, <lorenzo.pistone@kiratech.it>
