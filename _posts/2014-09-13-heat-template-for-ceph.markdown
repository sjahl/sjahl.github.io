---
layout: post
title: "Heat Template For Testing Ceph"
date: 2014-09-13 16:00:59
---

I put together an OpenStack [Heat][heat] template for bringing up a small 3-node cluster to test [Ceph][ceph]. Due to the lack of current Heat template documentation, I thought it'd be helpful to share, so you can find it in my [github repo][github].

With this template, you get three instances with one block device attached to each, prime for installing a small Ceph cluster for testing. The heat template outputs return the IPs of the three instances. When launching the stack, you'll need to provide a network UUID, the name (not UUID) of an image, and your nova keypair name. The rest of the options have defaults which you can leave, or set to the values of your choice. It's worth noting that your keypair gets installed into the ec2_user's home folder, rather than what you might be used to (I'm used to it being installed in the 'ubuntu' user's home folder, as is usually the case in the Ubuntu cloudimg).

I've tested this template with an Icehouse cloud running on a mix of Ubuntu precise and trusty compute hosts. Of course, you need Heat enabled on your cloud to use this.

### A few tips for reading the template

There are three main sections to a heat template:

1. The *parameters* section is where required values are placed. 

    The items in this section are what appear on the "Launch Stack" interface when booting a stack in horizon. Values are either defined statically, or defaults can be given, which auto-fill in the web interface. An item without a value or default produces an empty form field in the horizon interface, and you (or the user) are required to fill the value.

2. The *resources* section defines the various OpenStack pieces that make up the stack. 

    In the case of this template, we are defining three instances, volumes, attachments, and neutron ports. We can retrieve values from the parameters with the `get_param` keyword. Other resources in this section can be referenced with the `get_resource` keyword. This is how I've associated volume attachments between their respective volumes and instances.

3. The *outputs* section is for retrieving values that are displayed once the stack has been created. 

    I've used this for retrieving the IPs of the instances that have been launched (see the `get_attr` keyword) so that I don't have to go digging through the web interface for them -- they're all aggregated in the Stack Detail Overview in horizon for easy viewing.

That's all for now. Feel free to tell me that I've done it all wrong... or submit a pull request or something.


[heat]: https://wiki.openstack.org/wiki/Heat
[ceph]: http://ceph.com
[github]: https://github.com/sjahl/heat-templates/blob/master/templates/ceph-test-cluster.yaml