## How to patch nodes using Puppet

* Add `mod 'albatrossflavour-os_patching', '0.13.0'` to your Puppetfile and deploy your control repo
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

