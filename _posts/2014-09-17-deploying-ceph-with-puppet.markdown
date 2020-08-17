---
layout: post
title: "Deploying Ceph With Puppet"
date: 2014-09-18 08:30:00
---

Following an epic saga of hardware procurement, and about a week of unstable power and [networking at work][network], I managed to deploy a Ceph cluster this week. I did so using Puppet, and I figured I should write some of my process down, since the [current information][cephwiki] on the topic, is a bit light.

My setup includes:

* Ubuntu 14.04LTS
* Puppet 3.7 (+ hiera)
* The [stackforge/puppet-ceph][stackforge] module
* Ceph .80.5

To follow along, you'll need a few things:

* Hardware (or VMs) to install Ceph on
* Familiarity with Puppet and Hiera
* Familiarity with Ceph

## Puppet modules

I decided to go with the module on [stackforge][stackforge]. The only other module that I could see real evidence of people using in production was the [eNovance][enovance] one. Though, from what I could tell it wasn't clear what the future of that module was, and it seemed like there was some agreement that development efforts would converge on the module on stackforge.

I'm also somewhat of a fan of the Gerrit-based code review system in place for OpenStack, as it makes contributing to the modules fairly easy. Having the Ceph module hosted there means it can take advantage of that workflow. Worth noting, is that despite the module being hosted at stackforge, you can use it to deploy a Ceph cluster independently from an OpenStack cloud.

## Initial setup

You'll of course need to install the puppet-ceph module into your puppet modules tree. Also needed, will be a way to classify your nodes, such as with a puppet External Node Classifier (something like [Foreman][foreman]). Any way of getting puppet to associate roles with your nodes should do just fine.

### Hierarchy

Imagine we have a (simple) hiera hierarchy:

{% highlight yaml %}
:hierarchy:
- %{role}
- %{cluster}
- common
{% endhighlight %}

In my case, nodes are assigned to a "ceph" cluster during installation, and I later assign them a specific role (ceph-mon or ceph-osd) when I'm ready to finish provisioning them. Nodes in the "ceph" cluster have the `ceph::profile::base` and `ceph::repo` classes assigned to them. The base profile and repo classes will configure the ceph package repository, and install ceph.

In the hiera datadir, we'll end up with three files:

* ceph.yaml: Parameters that will apply cluster-wide
* ceph-mon.yaml: Parameters for the ceph monitor role
* ceph-osd.yaml: Parameters for the ceph OSD role


## Classes and variables

There's a few parameters that we'll need to provide to the module. I'll be using the [examples][examples] from the module for these values.

**ceph.yaml**

You should generate the `fsid` for your cluster with the `uuidgen -r` command. The `mon_initial_members` should be the short hostnames of the hosts you intend to use as monitors. `mon_host` should be set to the IPs of those monitors, specifying port 6789. The values for various keys can be defined with the `ceph-authtool --gen-print-key` command. We need to define the keys in the common file, so that they can be defined on the clients that need them, as well as injected into the cluster keyring by the monitor.

These should be defined in the hiera file applying to the "cluster", ceph.yaml in our case:

{% highlight yaml %}
ceph::profile::params:release: 'firefly' # the release you want to install
ceph::profile::params::fsid: '787b7f0f-3398-497a-bc9b-8fffb74f589e'
ceph::profile::params:authentication_type: 'cephx'
ceph::profile::params:mon_initial_members: 'first,second, third'
ceph::profile::params::mon_host: '10.11.12.2:6789, 10.11.12.3:6789, 10.11.12.4:6789'
ceph::profile::params::osd_pool_default_pg_num: '200'
ceph::profile::params::osd_pool_default_pgp_num: '200'
ceph::profile::params::osd_pool_default_size: '2'
ceph::profile::params::osd_pool_default_min_size: '1'
ceph::profile::params::cluster_network: '10.12.13.0/24'
ceph::profile::params::public_network: '10.11.12.0/24'

# keys
ceph::profile::params::mon_key: 'AQATGHJTUCBqIBAA7M2yafV1xctn1pgr3GcKPg=='
ceph::profile::params::admin_key: 'AQBMGHJTkC8HKhAAJ7NH255wYypgm1oVuV41MA=='
ceph::profile::params::admin_key_mode: '0600'
ceph::profile::params::bootstrap_osd_key: 'AQARG3JTsDDEHhAAVinHPiqvJkUi5Mww/URupw=='
ceph::profile::params::bootstrap_mds_key: 'AQCztJdSyNb0NBAASA2yPZPuwXeIQnDJ9O8gVw=='
{% endhighlight %}

**ceph-mon.yaml**

Next, we need to define the settings for servers that will be given the ceph-mon "role". There's actually not much we need to put into this file at the moment since keys and ceph.conf contents are defined in *ceph.yaml*. Just make sure your monitor servers have the `ceph::profile::mon` class applied to them. We'll return to this file later to add additional keys.

**ceph-osd.yaml**

Finally, a ceph-osd role needs to have the `ceph::profile::osd` class applied, and the OSD definitions themselves:

{% highlight yaml %}
ceph::profile::params::osds:
  '/dev/sdc':
    journal: '/dev/sdb1'
  '/dev/sdd':
    journal: '/dev/sdb2'
{% endhighlight %}

The above will provision a server with two OSDs, and will place the journal on two different (raw) partitions of an external journal disk.

## Ordering

I found that it worked best to install things in the following order:

1. Provision the servers, and apply the `ceph::profile::base` class to install Ceph and configure ceph.conf, etc.
2. Apply the ceph-mon role to the ceph monitors. While there were no OSDs in the cluster yet, I was able to log into the monitor nodes and test the `ceph status` command to ensure that I had a working monitor quorum, and that the monitor cephx keys were configured. Once the monitors are up, you can configure your CRUSH map (next step) before adding any OSDs.
3. Configure your CRUSH map. The default doesn't include many failure domains past 'host' buckets -- you'll definitely want to plan your CRUSH hierarchy for a production cluster. More details on that over at the [Ceph docs][crush-map-docs].
4. Apply the ceph-osd role to OSD servers. Puppet will format the disks specified in your `ceph::profile::params::osds` variable, and will automatically add them to the CRUSH map.

### OSD CRUSH location

Ceph provides a [config option][crush-map] for providing the CRUSH location of your OSDs. I found that it was a good idea to add this to the ceph.conf file before applying the ceph-osd role to my OSD nodes. The way the module is written, you can add values to ceph.conf on the host, and they won't be overwritten. With that option set, Puppet will automatically add the OSD to the desired location in your CRUSH map.

While doing this is manageable for a small cluster, it could be cumbersome if you're deploying 10s of storage nodes. Ceph alternatively allows you to specify a "osd crush location hook", which can be a custom script to specify the location string.

I suppose it would be nice if the puppet module allowed you to specify the crush location as well (maybe a sysadmin familiar with Ceph and Puppet should submit a patch ;-) )

## How Puppet handles OSD replacement

OSDs will eventually fail, and need to have their disk replaced. Luckily, Puppet can handle most of the dirty work for us! Simply replace the bad disk, and wait for puppet to provision it. However, there are some caveats with this.

When an OSD fails and gets marked "down" and "out" from the cluster, the OSD still remains in the CRUSH map. If you simply replace the disk and have puppet add it to the cluster, it will generate a new OSD in the map. In a cluster with 150 OSDs, it will add a 151st OSD to your CRUSH map, and leave the old downed OSD in the map. You can either remove the downed OSD after the new one is in place, or, you can remove it before replacing the disk, and puppet will add it back to the cluster with the original OSD number of the failed disk. I'm sure this is mostly cosmetic, but if having broken sequences of numbers triggers your OCD, it's something to think about.

In any case, removing an OSD from the CRUSH map looks like the following (assuming osd.34 is the dead disk). You'll need to run this on a host that has keys to the CRUSH map. Our monitors in this deployment have client.admin keys:

{% highlight bash %}
# make sure the OSD is out
ceph osd out osd.34
# remove it from the map
ceph osd crush remove osd.34
# delete it's authentication tokens
ceph auth del osd.34
# finally, remove the OSD
ceph osd rm osd.34
{% endhighlight %}

## A few notes on key management

If you plan to use your Ceph cluster with something else (such as OpenStack) which will need keys in order to speak to rados, we can define those keys in hiera. The module includes a `ceph::keys` class which makes it fairly trivial to provide keys to clients, and can inject them into the cluster keyring.

On the monitor host, we need to make sure the `ceph::keys` class is applied. We can provide a hash of keys to this class with the `args` variable, which looks like:

{% highlight yaml %}
ceph::keys::args:
  client.app:
    secret: (generate this as described above)
    cap_mon: allow r
    inject: true
    inject_as_id: mon.
    inject_keyring: /var/lib/ceph/mon/ceph-%{::hostname}/keyring
{% endhighlight %}

That will create a client.app keyring, and inject it to the cluster keyring using the mon. id. Now, in a hiera role that applies to our client, we'll need to apply the `ceph::profile::client` and `ceph::keys` classes to the client node:

{% highlight yaml %}
ceph::keys::args:
  client.app:
    secret: (use the same secret as defined on the monitor)
    cap_mon: allow r
{% endhighlight %}

That will write the keyring file to the default location on the client (/etc/ceph/ceph.$key_name)


Well, that's all for now folks. I hope this provides a good enough overview of the puppet-ceph module to get you started on your production deployments! If you notice an error, or want to tell me that I'm wrong, get at me on [Twitter][twitter].



[network]: http://blog.bimajority.org/2014/09/05/the-network-nightmare-that-ate-my-week/
[cephwiki]: http://wiki.ceph.com/Guides/General_Guides/Deploying_Ceph_with_Puppet
[stackforge]: https://github.com/stackforge/puppet-ceph
[enovance]: https://github.com/enovance/puppet-ceph
[examples]: https://github.com/stackforge/puppet-ceph/blob/master/examples/common.yaml
[foreman]: http://theforeman.org
[crush-map]: http://ceph.com/docs/master/rados/operations/crush-map/#crush-location
[crush-map-docs]: http://ceph.com/docs/master/rados/operations/crush-map
[twitter]: https://twitter.com/roguetortoise