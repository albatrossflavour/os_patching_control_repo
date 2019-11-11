# puppet_os_patching module blog post

## Who am I?
I have been a UNIX systems administrator for over 25 years, in those times servers weren't pets, they were more like children.  My goal has always been to automate myself into obsolescence and Puppet is the tool I've been using to make that happen.

I have worked in the finance, telecommunications and media industries, where I have helped develop people, tools and services.  I am now leading the DevOps practice for Katana 1, a Puppet partner in Sydney Australia.

## What was the problem?
In my years of being a sysadmin, patching was always the bane of everyone's existence.  Patching is manually intensive, centralised information almost never exists, it requires a lot of coordination to be done to arrange downtime and the processes vary wildly between different operating systems, distributions and even releases of distributions.

### But what is patching?
Establishing a common understanding of what patching means is important.  [There are](https://en.wikipedia.org/wiki/Patch_(computing)) lots of different interpretations out there, but for our purposes, this is what **patching** means.

> Applying changes to computer software with the intention of resolving functional or security bugs, improving usability, reliability or performance.

### Pain points
Unpatched vulnerabilities are [obviously a huge concern](https://www.infosecurity-magazine.com/opinions/unpatched-vulnerabilities-cause/) and put organisations at significant risk.  Even if you know about a vulnerability, tracking down affected servers either requires manual effort, specialised tools, custom tool development, or a combination of the above.

Once you do have a list of affected servers, how do you go about actually applying the updates?  Centralised control of the process by an IT team is a common approach and this can work well, however there are some areas where this falls short.  Picking patching windows, preventing a server from being patched and controlling if all patches or just security ones should be applied.

Self service options solve *some* of those issues, but open up others.  Training, access control and enforcement of standards come to mind.

Once patching is complete, you need to validate success and ensure your reporting systems are updated with the new state.

Ensuring all stakeholders have access to the patch state of the servers and having that data be timely and accurate.

If it is such a risk, why is it still so hard?

### ... wait a minute, I have an idea

I already have a location where I can [centralise data](https://docs.puppet.com/puppetdb/), a [way of keeping data accurate](https://glennsarti.github.io/blog/puppet-ruby-facts/), an [RBAC](https://puppet.com/products/capabilities/role-based-access-control-rbac) system and the ability to [trigger adhoc work on nodes](https://puppet.com/docs/bolt/latest/writing_tasks_and_plans.html).  Not only that, most of what I need can already be driven by and reported on though an [API](https://puppet.com/docs/pe/latest/api_index.html) and a [web console](https://puppet.com/docs/pe/latest/console_accessing.html).  So what was I waiting for?

I immediately started work on what eventually became the `os_patching` module.

The module ***currently*** is fully functional on Linux (RedHat and Debian).  Support for other OS types, specifically Windows, is being actively worked on.

## What did it need to do?

* Report the patch state on a server, via custom facts, back into PuppetDB
* Show which updates are related to security (if possible)
* Allow the servers to be assigned to a 'patch window' to allow simple scheduling
* Allow blackout times to be set for servers which would prevent any patching activity
* Ability to control if a post-patching reboot is performed
* Enable the execution of a 'patch run' on a defined group of servers, including:
    * being able to clean package caches
    * restricting to security patching
    * supplying overrides for the arguments to the OS package commands
    * being able to trigger the patch run from the command line, console or through an API
    * have control on who can execute a patch run
* Store the canonical patching state data on the node

The last item was one of the most important.  No matter what happened, I wanted to be sure that the facts on the node the source of truth for patching information and everything else was fed from there.

## How does it do it?

To get started with the module, you simply need to declare the `os_patching` module onto your nodes.  It will setup a scheduled task to refresh the patch information and allow access to the tasks to carry out the patching.

### "Just the facts ma'am"

Custom facts underpin the entire module, so making sure we have a good structure and management system for them was key.

```json
{
  warnings => {},
  package_updates => [
    "glibc.x86_64",
    "glibc-common.x86_64",
    "kernel.x86_64",
    "kernel-tools.x86_64",
    "kernel-tools-libs.x86_64",
    ...
    "util-linux.x86_64"
  ],
  package_update_count => 32,
  security_package_updates => [],
  security_package_update_count => 0,
  blocked => false,
  blocked_reasons => [],
  blackouts => {
    End of year change freeze => {
      start => "2018-12-15T00:00:00+10:00",
      end => "2019-01-01T00:00:00+10:00"
    }
  },
  pinned_packages => [],
  last_run => {},
  patch_window => "",
  reboot_override => "default",
  reboots => {
    reboot_required => false,
    apps_needing_restart => {},
    app_restart_required => false
  }
}
```

The custom facts pull their values from cached values, some are generated by a scheduled script/task and others by the `os_patching` class.  We cache these values as having hundreds of nodes hitting your apt/yum servers every time they run facter would cause some pretty big issues.  The scheduled script, by default, refreshes the cache data every hour, however this can be overridden through classification.

The facts are broken down into a couple of sections

#### State facts

These facts show the state of the node.

* Are there patches to apply?
* If so, how many?
* Are they security related?
* Does the node need to be rebooted?
* Do apps need to be restarted?
* Is patching blocked?

#### Control facts

These facts control the execution of patching.

* Do we have blackout windows defined?
* Is the node allocated to a patching window?
* Should this node override the reboot parameter?


The facts give us access to all of the information we need to audit the fleet and control the patch runs.

Nodes can be assigned to a `patch_window` to group them (think "Group 1", "Week 4").  Blackout windows can be defined for change freezes or for nodes which cannot be patched.

When you combine all of these facts, you can write queries such as:

`puppet task run os_patching::patch_server --query='nodes[certname] { facts.os_patching.patch_window = "Week3" and facts.os_patching.package_update_count > 0 and facts.os_patching.blocked = false }'`

This task would patch any node that is assigned to the patch window "Week3", is not blocked, and that has patches waiting to apply.


#### Controlling the facts

There are a number of other settings you can configure if you'd like.

* patch_window: a string descriptor used to “tag” a group of machines, i.e. Week3 or Group2
* blackout_windows: a hash of date-time start/end dates during which updates are blocked
* security_only: boolean, when enabled only the security_package_updates packages and dependencies are updated
* reboot_override: boolean, overrides the task's reboot flag (default: false)
* dpkg_options/yum_options: a string of additional flags/options to dpkg or yum, respectively

You can set these in hiera. For instance, my global config has some blackout windows for the next few years:

```
os_patching::blackout_windows:
  'End of year 2018 change freeze':
    'start': '2018-12-15T00:00:00+1000'
    'end':   '2019-01-05T23:59:59+1000'
  'End of year 2019 change freeze':
    'start': '2019-12-15T00:00:00+1000'
    'end':   '2020-01-05T23:59:59+1000'
  'End of year 2020 change freeze':
    'start': '2020-12-15T00:00:00+1000'
    'end':   '2021-01-05T23:59:59+1000'
  'End of year 2021 change freeze':
    'start': '2021-12-15T00:00:00+1000'
    'end':   '2022-01-05T23:59:59+1000'
```

## Actually do the patching

Since I was going to be using tasks, I didn't have to worry about how to implement RBAC or how to trigger the patch run.

I setup 3 tasks:

* `clean_cache` : cleans the package cache on the nodes (`yum clean all` for example)
* `refresh_fact` : forces a regeneration of the patching cache data
* `patch_server` : actually runs the patching

The `patch_server` task is what we'll look at in more detail now.

When triggered, the task will first check the value of the fact `os_patching.blocked`.  If it is set to true, the task exits as there is a reason that the patching cannot continue.  This would usually mean that the node is within a blackout window.

Providing there are patches to apply, the task then kicks off the OS patching command under a timeout value (3600 seconds by default).  It waits for completion and then pulls back the job information for reporting.

The facts are then refreshed and pushed up to the puppetserver (`puppet fact upload`) then we enter **Reboot Town**.

## To reboot or not to reboot

Remember, patching the node does no good unless you restart the processes which were using the affected packages.  This could be as simple as an application restart or as invasive as a full reboot.  So how do we control that?

You can control what reboot action the task will take by using the `reboot` parameter.  It accepts the following values:

* `always` : Irrespective of what happened during the task, reboot the node.  This will ***ALWAYS*** trigger a reboot
* `never` : Irrespective of what happened during the task, do not reboot the node.  This will ***NEVER*** trigger a reboot
* `patched` : Trigger a reboot if any patches were applied
* `smart` : Use the OS tools (`needs-restarting` on RedHat,  `/var/run/reboot_required` on Debian) to determine if a reboot is required after patching.  This will only trigger a reboot if kernel/core libraries were updated.

You can also use the fact `os_patching.reboot_override` to customise behaviour on a granular level, such as having all nodes set to reboot other than three which are set to `never` as you know they will be rebooted manually at a later date.


## Flowchart
This is all a bit complex, might be easier in a flow chart.

![](http://bandcamp.tv/puppet-blog-images/flowchart.png)

## Output

The following is a sample of the task output, visible from the command line, through the console, or through the API.
```
{
  "pinned_packages" : [ ],
  "security" : false,
  "return" : "Success",
  "start_time" : "2019-03-20T01:15:07+11:00",
  "debug" : "Loaded plugins: fastestmirror\nLoading mirror speeds from cached hostfile\n ... 8< SNIP >8 ...",
  "end_time" : "2019-03-20T01:16:47+11:00",
  "reboot" : "smart",
  "packages_updated" : {
    "openssl-1:1.0.2k-16.el7.x86_64" : "Updated",
    "polkit-0.112-18.el7.x86_64" : "Updated",
    "kernel-tools-libs-3.10.0-957.5.1.el7.x86_64" : "Updated",
    "openssl-libs-1:1.0.2k-16.el7.x86_64" : "Updated",
    "rpm-4.11.3-35.el7.x86_64" : "Installed",
    "yum-3.4.3-161.el7.centos.noarch" : "Installed",
    "yum-plugin-fastestmirror-1.1.31-50.el7.noarch" : "Installed",
    "kernel-tools-3.10.0-957.5.1.el7.x86_64" : "Updated",
    "kernel-3.10.0-957.10.1.el7.x86_64" : "Install",
    "python-perf-3.10.0-957.5.1.el7.x86_64" : "Updated"
  },
  "job_id" : "13",
  "message" : "Patching complete"
}
```

The entries are:

* `pinned_packages` : any packages version locked/pinned at the OS layer
* `debug` : full output from the patching command
* `start_time/end_time` : when the task started/stopped
* `reboot` : the reboot parameter used
* `packages_updated` : which packages were affected
* `security` : the security parameter used
* `job_id` : the yum job ID (only populated on RedHat family nodes)
* `message` : status info

## TL;DR

There is a lot of info above, but you might just want to get started with using the `os_patching` module, so here are the steps.

* Add `mod 'albatrossflavour-os_patching', '0.8.0'` to your Puppetfile and deploy your control repo
* Classify the linux nodes you wish to be able to patch with the `os_patching` module
    * `include os_patching` within a [patching profile](https://github.com/albatrossflavour/puppet_os_patching/blob/development/examples/sample_patching_profile.pp)
    * Use the PE console ![](http://bandcamp.tv/puppet-blog-images/classification.png)
* Run puppet on these nodes and expect the following changes:
    * The file `/usr/local/bin/os_patching_fact_generation.sh` will be installed
    * Cron jobs will be setup to run the script every hour (using [fqdn_rand](https://puppet.com/docs/puppet/5.4/function.html#fqdnrand)) and at reboot
    * The directory `/var/cache/os_patching` will be created
    * `/usr/local/bin/os_patching_fact_generation.sh` will run and will populate files into `/var/cache/os_patching`
    * A new fact (`os_patching`) will be available
* View the contents of the `os_patching` fact on the nodes you classified:
    * `facter -p os_patching`
    * `puppet-task run facter_task fact=os_patching --nodes centos.example.com`
    * Use the console to view the fact ![](http://bandcamp.tv/puppet-blog-images/facts.png)
* Execute a patch run on these nodes:
    * `puppet task run os_patching::patch_server --query='nodes[certname] { facts.os_patching.package_update_count > 0 and facts.os_patching.blocked = false }'`
    * Run the task through the console ![](http://bandcamp.tv/puppet-blog-images/task.png)
