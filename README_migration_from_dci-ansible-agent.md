# Migration from DCI Ansible Agent

The DCI OpenStack Agent used to be named `dci-ansible-agent`. The name was
confusing we we decided to rename it `dci-openstack-agent`. To transition to
the new package you have to follow a couple of manual steps.

First, install the new agent:

    # yum install -y dci-openstack-agent

You must now disable the `dci-ansible-agent`:

    # systemctl stop dci-ansible-agent.timer
    # systemctl stop dci-ansible-agent
    # systemctl disable dci-ansible-agent.timer
    # systemctl disable dci-ansible-agent

Now you can rename the configuration files:

    # cp -Rv /etc/dci-ansible-agent/* /etc/dci-openstack-agent/
    # sed -i 's,/dci-ansible-agent",/dci-openstack-agent",' /etc/dci-openstack-agent/settings.yml
    # cp -Rv /var/lib/dci-ansible-agent/.ssh /var/lib/dci-openstack-agent/.ssh
    # cp -Rv /var/lib/dci-ansible-agent/*.tar /var/lib/dci-openstack-agent/
    # restorecon -R /var/lib/dci-openstack-agent/
    # chown -R dci-openstack-agent:dci-openstack-agent /var/lib/dci-openstack-agent/

Finally you can mask the old service restart the agent with the new one:

    # systemctl mask dci-ansible-agent.system
    # systemctl start dci-openstack-agent.timer
    # systemctl start dci-openstack-agent

You do not need to manually remove the `dci-ansible-agent` rpm. It will be
automatically remove by an update in the future.
