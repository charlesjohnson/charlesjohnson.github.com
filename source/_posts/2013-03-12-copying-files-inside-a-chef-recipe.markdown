---
layout: post
title: "Copying Files Inside A Chef Recipe"
date: 2013-03-12 16:54
comments: true
categories: 
- Chef
- Vagrant
- chef-client
- tableflip
---

It's a little puzzling to me that while Chef has the [remote_file](http://docs.opscode.com/chef/resources.html#remote_file) resource for copying files from remote hosts to local, it doesn't have a similar mechanism in the [file](http://docs.opscode.com/chef/resources.html#remote_file) resource to allow for copying files from one location on the local host to another as part of a recipe. As a deploy pattern unnecessary file copies should be discouraged, but sometimes it's just necessary to copy a file once it's been placed on the system at another location by an outside actor. One good example of this is when using a [Vagrant](http://www.vagrantup.com) guest with the [Chef-Server](http://docs-v1.vagrantup.com/v1/docs/provisioners/chef_server.html) provisioner and the [chef-client](http://community.opscode.com/cookbooks/chef-client) cookbook.

When Vagrant starts a run using the Chef-Server provisioner, it writes out a `client.rb` file to the guest at `/tmp/vagrant-chef-1/client.rb`, and then starts a single run of chef-client with the `-c /tmp/vagrant-chef-1/client.rb` command-line flag, telling it to look for the config file in the irregular location. When run without that flag, chef-client looks for the file at the default location, `/etc/chef/client.rb`.

But what if the recipe being tested with Vagrant adds `recipe[chef-client::service]` to the node's run_list? Vagrant will start an instance of chef-client, using the Vagrant-generated client.rb file at `/tmp/vagrant-chef-1/client.rb`. When this copy of chef-client runs the chef-client::service recipe, the Vagrant-instantiated copy of chef-client will try to start a second, daemonized copy of chef-client as a service, this time without the -c flag. The second copy of chef-client will look for the config file at `/etc/chef/client.rb`, but because the config is not there, the provisioner will exit with an error.

This makes it tricky to test code that install Chef-client as a service using Vagrant, because Vagrant writes out the `client.rb` file to /tmp, which isn't a suitable place to hold long-term data in any environment. Unlike Vagrant's [Chef-Solo](http://docs-v1.vagrantup.com/v1/docs/provisioners/chef_solo.html) provisioner, it's impossible to inject node attributes directly via the Vagrantfile with the Chef-Server provisioner, so we can't easily write an exception into the Vagrantfile itself. Either a more complicated workflow must be devised to set the `["chef_client"]["conf_dir"]` attribute to point to `/tmp/vagrant-chef-1`, or (more simply) the client.rb file should be copied to `/etc/chef/client.rb` prior to running the `chef-client::service` recipe.

Here's a code block that should take care of this use case, with values hard-coded for readability:

```ruby
if ::File.exist?("/tmp/vagrant-chef-1/client.rb")
  ruby_block "copy Chef config file if running in a Vagrant guest" do
   block do
    ::FileUtils.cp "/tmp/vagrant-chef-1/client.rb", "/etc/chef/client.rb"
   end
   if ::File.exist?("/etc/chef/client.rb")
     not_if { ::FileUtils.compare_file("/tmp/vagrant-chef-1/client.rb", "/etc/chef/client.rb") }
   end
  end
end
```

What this does: First check to see if the source file exists. If it does, check to see if the target file exists. If both files exist, then compare them to see if they are identical. If the files are not identical, or if the target file does not exist, then copy the source file to the target file location, overwriting the target file if present. In all other cases, do nothing.

Drop this in before the `chef-client::service` recipe, and the Vagrant provision run will now complete, because the Chef-client service will be able to start.


One note: While I haven't done any direct testing, it looks like Ruby's [FileUtils.compare_file()](http://ruby-doc.org/stdlib-1.9.3/libdoc/fileutils/rdoc/FileUtils.html#method-c-compare_file_) method works by streaming both files and comparing them, so it should work with binaries. However, I could imagine this being very memory-hungry if attempted with large files. Or maybe not? I'm a lousy Rubyist.
