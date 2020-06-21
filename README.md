# Ansible configuration optimization

A handful of settings I like to add to each project's `ansible.cfg` INI file for better performance, readability, and diagnostic information.

## Performance

### Pipelining

This allows batching of SSH requests and less connection time—a significant performance gain. `requiretty` must be disabled in the target host's `/etc/sudoers` file for this to work, but this should be the default in CentOS 7 and many other distributions and shouldn't cause problems. Test and see.

This goes under the `[ssh_connection]` section of the INI:

```
[ssh_connection]
pipelining = true
```

### Use Mitogen instead

The third-party Mitogen project converts Ansible's connections and module runtime into pure Python, for a much faster and more performant experience.

In testing across a range of modules, I've only experienced Mitogen errors or module compatibility on only a couple of occasions—otherwise, it is production-ready, and quite effective, reducing one 13-minute playbook (with pipelining on) to four minutes, and is a touch faster than pipelining running my standard WordPress-and-LEMP playbook on a local Vagrant CentOS VM. 

It will not make package installation faster, but it absolutely races through tasks such as file configuration—playbooks heavy on this, or with many small tasks, should be much faster. Post-installation runs with Mitogen should be notably faster than ones without.

Download and unzip the Mitogen for Ansible tarball from [the Mitogen website](https://mitogen.networkgenomics.com/ansible_detailed.html) and put it somewhere safe, then update your playbook:

```
[defaults]
strategy_plugins = /path/to/mitogen-0.2.9/ansible_mitogen/plugins/strategy
strategy = mitogen_linear
```  

## Track playbook task timing

Measure your performance improvements with the `profile_tasks` or `profile_roles` callback plugin, which goes under `[defaults]`

```
[defaults]
callback_whitelist = profile_tasks
```

This gives a nice read-out of playbook actions during the run and a summary at the end.

## See only changed tasks and errors

Skip the output for "OK" and skipped tasks. I usually toggle this on after the first few times a playbook runs for cleaner output. Especially nice for a long playbook.

This was previously activated by setting the `stdout_callback` to `actionable`:

```
[defaults]
stdout_callback = actionable
```

But this will be deprecated in Ansible 2.11, as the default stdout plugin can do it with these settings, which work in the current Ansible version (tested by me on 2.9+):

```
[defaults]
display_skipped_hosts = no
display_ok_hosts = no
```

With `profile_tasks` running, you'll still get timings on the OK and skipped tasks, but without the verbosity of the task names and status. 

## Output colors

Ansible's default colors are sensible to me:
- green for "OK" messages
- red for errors
- yellow for changes
- blue for skipped tasks

But running `--diff` on a playbook gives us green and red again for added and replaced text, which is confusing to me and hard to pick out in a sea of like colors.

We can change these (and the other defaults) under `[colors]`:

```
[colors]
diff_add = bright purple
diff_remove = bright yellow
```

The list of [supported Ansible colors](https://github.com/ansible/ansible/blob/devel/lib/ansible/utils/color.py) is in the Ansible codebase. 

"Bright" and "dark" colors come out simply as bold in my terminal, which helps. Purple gets my attention, and the default use of yellow for change is more natural than red. "Dark gray" also works for me here, but it would be nice to have orange.

## Show colors in CI/CD

Make sure to show colors instead of a plain black or white read-out in GitLab, etc., by setting:

```
[colors]
force_color = True
```

You can also set this as an environment variable, which is nice for GitLab: `ANSIBLE_FORCE_COLOR=True`

## Set a custom inventory

I like to do this to keep a cleaner project root, by adding an `inventory.ini` file inside an `inventory` folder instead of using the expected default `hosts` file. 

```
[defaults]
inventory = inventory/inventory.ini
```

## Set a custom SSH config

Rather than setting up SSH values and arguments as Ansible variables, you can just add a config file to your project and write them in the standard syntax. I usually add these in `group_vars` instead, but it's a neat trick.

```
[ssh_connection]
ssh_args = -F .ssh/config
```

## More Resources

- A helpful set of [official Ansible examples](https://raw.githubusercontent.com/ansible/ansible/devel/examples/ansible.cfg) and defaults to be aware of
- List of [Ansible configuration options](https://docs.ansible.com/ansible/latest/reference_appendices/config.html) and environment variables from the Ansible documentation


