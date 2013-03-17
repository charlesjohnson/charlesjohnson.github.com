---
layout: post
title: "Copying Files as Part of a Chef Recipe"
date: 2013-03-12 16:54
comments: true
categories: 
- Chef
- Vagrant
- chef-client
- tableflip
---

It's a little puzzling to me that while Chef has the [remote_file](http://docs.opscode.com/chef/resources.html#remote_file) resource for downloading files from remote hosts, it doesn't have a similar mechanism in the [file](http://docs.opscode.com/chef/resources.html#remote_file) resource to copy files as part of a recipe. While there's usually a workaround and unnecessary file copies should be discourage in a deploy or configuration pattern, sometimes outside actors beyond our control may place an essential file at an inconvenient and non-configurable location on the filesystem.

One good example of this occurs when a [Vagrant](http://www.vagrantup.com) guest with the [Chef-Server](http://docs-v1.vagrantup.com/v1/docs/provisioners/chef_server.html) provisioner has the [chef-client](http://community.opscode.com/cookbooks/chef-cent) cookbook in the [run_list](http://docs.opscode.com/essentials_node_object_run_lists.html). When Vagrant invokes the Chef-Server provisioner, it writes out a `client.rb` file to the guest filesystem at `/tmp/vagrant-chef-1/client.rb`, along with a few others. The provisioner then runs chef-client in the guest VM with the `-c /tmp/vagrant-chef-1/client.rb` command-line flag, redirecting chef-client to the config file at this location.

```bash
/tmp/vagrant-chef-1
|-- client.rb
|-- dna.json
`-- validation.pem
```

By contrast, when chef-client is started as a daemon by the `chef-client::service` recipe, chef-client looks for configuration at the location `#{node['chef-client']['conf_dir']}/client.rb`, which defaults to `/etc/chef/client.rb.` So what happens if the recipe being tested with Vagrant adds `recipe[chef-client::service]` to the node's run_list?<!--more-->

[{% img center http://i.imgur.com/wMT8GvR.png 576 539 'Chef provisioner flowchart' 'A flowchart diagram of Vagrant running chef-client with the chef-server provisioner' %}](http://i.imgur.com/wMT8GvR)

The Vagrant provisioner will generate a client.rb file at `/tmp/vagrant-chef-1/client.rb`, and then start an instance of chef-client using this configuration file. When the provisioning instance of chef-client converges the `chef-client::service` recipe, it will try to start a second, daemonized copy of chef-client via an init script. This second instance of chef-client will look for configuration at the location `#{node['chef-client']['conf_dir']}/client.rb`, which defaults to `/etc/chef/client.rb`. But because there is no config file at this location, the service will not start, the provisioning chef-client run will exit with an error, and the vagrant provision will fail.

It's tricky to use Vagrant to test code that starts chef-client as a service during the provisioner. To fix this issue, we have to either tell the daemonized instance of chef-client to look for the configuration file in another location, or we have to copy the configuration file to the location that the daemon is expecting. It turns out that redirecting the daemon by changing the attribute isn't the best choice: Unlike Vagrant's [Chef-Solo](http://docs-v1.vagrantup.com/v1/docs/provisioners/chef_solo.html) provisioner, it's not possible to inject node attributes directly into the Chef-Server provisioner's run via the Vagrantfile. In addition, because Vagrant writes out the `client.rb` file to `/tmp/vagrant-chef-1/client.rb`, setting the attribute to this location would set an uncomfortable (and unwise) precedent. So instead, the best solution is arguably to wait until Vagrant writes out the configuration file at `/tmp/vagrant-chef-1/client.rb`, and then copy it to `/etc/chef/client.rb` before attempting to converge the `chef-cent::service` recipe.

This code should should take care of this use case. It assumes that the attribute `node['chef-client']['conf_dir']` has been set.

```ruby
if ::File.exist?("/tmp/vagrant-chef-1/client.rb")
  ruby_block "Copy Chef config file if running in a Vagrant guest" do
   block do
    ::FileUtils.cp "/tmp/vagrant-chef-1/client.rb", "#{node['chef-client']['conf_dir']}/client.rb"
   end
   if ::File.exist?("#{node['chef-client']['conf_dir']}/client.rb")
     not_if { ::FileUtils.compare_file("/tmp/vagrant-chef-1/client.rb", "#{node['chef-client']['conf_dir']}/client.rb") }
   end
  end
end
```

What this does: First check to see if the source file exists. If it does, check to see if the target file exists. If both files exist, then compare them to see if they are identical. If the files are not identical, or if the target file does not exist, then copy the source file to the target file location, overwriting the target file if present. In all other cases, do nothing.

Drop this in a recipe that's included before the `chef-client::service` recipe as part of the run_list in the Vagrantfile, and the Vagrant provision run will now complete. **Be careful, there are security implications here!** Because `/tmp` is typically world-writeable, we can't rely on Vagrant being the only outside actor to leave a client.rb at this location. Proceed with caution.

One note: While I haven't done any direct testing, it looks like Ruby's [FileUtils.compare_file()](http://ruby-doc.org/stdlib-1.9.3/libdoc/fileutils/rdoc/FileUtils.html#method-c-compare_file_) method works by streaming both files and comparing them, so it should work with binaries. However, I could imagine this being very memory-hungry if attempted with large files. Or maybe not? I'm a lousy Rubyist.
