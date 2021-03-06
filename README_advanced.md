# DCI OpenStack Agent Advanced

## How to deal with multiple OpenStack releases

When testing multiple OpenStack releases you probably have different steps
(configuration, tasks, packages, etc...) according to the release. As an example
 you could:

- have scripts per OpenStack versions and file path based on the dci_topic
 variable (ie OSP10, OSP11, etc..):

```yaml
- shell: |
    /automation_path/{{ dci_topic }}/undercloud_installation.sh
```

- have git branch per OpenStack versions based on the dci_topic variable :

```yaml
- git:
    repo: https://repo_url/path/to/automation.git
    dest: /automation_path
    version: '{{ dci_topic }}'

- shell: /automation_path/undercloud_installation.sh
```

- use ansible condition and jinja template with the dci_topic variable :

```yaml
- shell: |
    /automation_path/build_container.sh
  when: dci_topic in ['OSP12', 'OSP13']

- shell: >
    source /home/stack/stackrc &&
    openstack overcloud deploy --templates
    {% if dci_topic in ['OSP12', 'OSP13'] %}
    -e /usr/share/openstack-tripleo-heat-templates/environments/docker.yaml
    -e /usr/share/openstack-tripleo-heat-templates/environments/docker-ha.yaml
    {% endif %}
    -e /usr/share/openstack-tripleo-heat-templates/environments/disable-telemetry.yaml
```

## How to retrieve the OpenStack yum repository

During the 'new' hook, the jumpbox will create a yum repository with latest bits
 available. This repository is located in the `/var/www/html/dci_repo` directory
 and accessible via HTTP at `http://$jumpbox_ip/dci_repo/dci_repo.repo`.

There's several ways to retrieve the yum repository from the undercloud:

- Using the yum-config-manager command:

```yaml
- shell: |
    yum-config-manager --add-repo {{ dci_baseurl }}/dci_repo/dci_repo.repo
  become: true
```

- Using the http url:

```yaml
- get_url:
    url: '{{ dci_baseurl }}/dci_repo/dci_repo.repo'
    dest: /etc/yum.repos.d/dci_repo.repo
  become: true
```

- Using the ansible copy module:

```yaml
- copy:
    src: /var/www/html/dci_repo/dci_repo.repo
    dest: /etc/yum.repos.d/dci_repo.repo
  become: true
```

## How to fetch and use the images

If you are use OSP12 and above, the DCI agent will set up an image registry and
 fetch the last OSP images on your jumpbox.

Before you start the overcloud deploy with the `openstack overcloud deploy
 --templates [additional parameters]` command, you have to call the following
 command on the undercloud node:

```console
$ openstack overcloud container image prepare --namespace ${jump_box}:5000/rhosp12 --output-env-file ~/docker_registry.yaml
```

:information_source: `${jump_box}` is the IP address of the Jumpbox machine and
 in this example we assume you use OSP12.

You don't have to do any additional `openstack overcloud container` call unless
 you want to rebuild or patch an image.

The Overcloud deployment is standard, you just have to include the two following
 extra Heat template:

- `/usr/share/openstack-tripleo-heat-templates/environments/docker.yaml`
- `~/docker_registry.yaml`

See the upstream documentation if you need more details: [Deploying the containerized Overcloud](https://docs.openstack.org/tripleo-docs/latest/install/containers_deployment/overcloud.html#deploying-the-containerized-overcloud)

Starting with OSP14 (Rocky), the undercloud is now also containerized. That means
that you also need to generate the container image list before the undercloud
installation.

```console
$ openstack overcloud container image prepare --namespace ${jump_box}:5000/rhosp14
                                              --roles-file /usr/share/openstack-tripleo-heat-templates/roles_data_undercloud.yaml
                                              --output-env-file ~/docker_registry.yaml

```

Finally specify the generated file path in the undercloud configuration and add
the jumpbox ip in the list of the docker insecure registries:

```console
$ vim undercloud.conf
```

```ini
[DEFAULT]
container_images_file = /home/stack/docker_registry.yaml
docker_insecure_registries = ${jump_box}:5000
```

## How to skip downloading some container images

Each Openstack release comes with +100 container images.
In most of the cases you don't need all the images to do the deployment because
 some are specific to extra services (barbican, manila, sahara, etc...)
If you want to skip downloading some images, you need to add the list of the
 associated openstack services in the settings.yaml file.

```yaml
skip_container_images:
  - barbican
  - manila
  - sahara
  - swift
```

## How to run the Update and Upgrade

After the deployment of the OpenStack, the agent will look for an update or an
 upgrade playbook. If the playbook exists it will run it in order to upgrade the
 installation.

The agent expects the upgrade playbook to have the following naming convention:

`/etc/dci-openstack-agent/hooks/upgrade_from_OSP9_to_OSP10.yml`

In this example, `OSP9` is the current version and `OSP10` is the version to
 upgrade to. Here is an example of an update playbook:

`/etc/dci-openstack-agent/hooks/update_OSP9.yml`

During the upgrade, you may need to use a specific version of a repository.
Each component has its own .repo. They are located in
<http://$jumpbox_ip/dci_repo/>, for instance:
<http://$jumpbox_ip/dci_repo/dci_repo_RH7-RHOS-11.0.repo>.

## How to run my own set of tests ?

`dci-openstack-agent` ships with a pre-defined set of tests that will be run. It
 is however possible for anyone, in addition of the pre-defined tests, to run
 their own set of tests.

In order to do so, a user needs to drop the tasks to run in `/etc/dci-openstack-agent/hooks/local_tests.yml`.

**NOTE**: Tasks run in this playbook will be run from the undercloud node. To
 have an improved user-experience in the DCI web application, the suite should
 ideally returns `JUnit` formatted results. If not in JUnit, one will be able to
 download the results but not see them in the web interface directly.

## How to adjust the timer configuration

```console
# systemctl edit --full dci-openstack-agent.timer
```

You have to edit the value of the `OnUnitActiveSec` key. According to systemd
 documentation:

```
OnUnitActiveSec= defines a timer relative to when the unit the timer is activating was last activated.
OnUnitInactiveSec= defines a timer relative to when the unit the timer is activating was last deactivated.
```

DCI comes with a default value of 1h, you can increase to 12h for example.

## Debug: How to manually run the agent

You may want to trace the agent execution to understand a problem. In this case,
 you can call it manually:

```console
# su - dci-openstack-agent -s /bin/bash
$ cd /usr/share/dci-openstack-agent
$ source /etc/dci-openstack-agent/dcirc.sh
$ /usr/bin/ansible-playbook -vv /usr/share/dci-openstack-agent/dci-openstack-agent.yml -e @/etc/dci-openstack-agent/settings.yml
```

## Tempest: How to disable services

The agent installs by default the meta tempest package openstack-tempest-all
 which contains all the tempest plugin tests. If you run tempest with the
 default configuration, you will execute tests on services that you probably
 don't want because the service isn't install.

In the tempest configuration you can disable tests per service. Each service has
 a boolean entry under [the service_available section](https://github.com/openstack/tempest/blob/master/tempest/config.py#L975-L994).

You can use the tempest_extra_config variable in the settings.yml file to add
 some services to disable:

```console
$ vim /etc/dci-openstack-agent/settings.yml

tempest_extra_config:
(...)
  service_available.designate: False
  service_available.ironic: False
  service_available.sahara: False
```

## Tempest: How to customize services

Depending on the drivers (cinder, manila, neutron, etc...) you are using, you
 will probably need to adapt the tempest configuration for specific tests. You
 can enable/disable features per OpenStack services. Every service has a section
 with the suffix '-feature-enabled' that allows to enable or disable features.
 If your cinder driver doesn't support cinder backup and you don't deploy the
 service, then you can disable the feature to avoid tempest failures.

```console
$ vim /etc/dci-openstack-agent/settings.yml

tempest_extra_config:
(...)
  volume-feature-enabled.backup: False
```

You can also do specific configuration per OpenStack services.
For instance, the cinder volume type tests are using the storage_protocol
 ('iSCSI') and vendor_name ('Open Source') fields that are configured by the
 storage driver. The default values in tempest don't match those in all storage
 drivers that's why you need to adapt the tempest configuration per services if
 you don't want some false positive. As an example, if you're using Ceph as a
 cinder backend you will need to update those values like:

```console
$ vim /etc/dci-openstack-agent/settings.yml

tempest_extra_config:
(...)
  volume.storage_protocol: 'ceph'
```

You will have some k/v with the 'network' prefix for neutron, 'compute' prefix
 for nova, etc... You can find most of the tempest config items in [the tempest project config](https://github.com/openstack/tempest/blob/master/tempest/config.py)

## Tempest: Run a given test manually

It may be useful to restart a failing test to troubleshoot the problem:

```console
$ /home/stack/tempest
$ ostestr --regex tempest.api.network.test_ports.PortsIpV6TestJSON.test_update_port_with_security_group_and_extra_attributes
```

The Certification test-suite uses it's own configuration located at `/etc/redhat-certification-openstack/tempest.conf`. Is a copy of `/home/stack/tempest/etc/tempest.conf`.

## How to test several versions of OpenStack

You can off course run different versions of OpenStack with the same jumpbox.
 To do so, you need first to adjust the way systemd call the agent:

```console
# systemctl edit --full dci-openstack-agent
```

Content :

```ini
[Unit]
Description=DCI Ansible Agent

[Service]
Type=oneshot
WorkingDirectory=/usr/share/dci-openstack-agent
EnvironmentFile=/etc/dci-openstack-agent/dcirc.sh
ExecStart=-/usr/bin/ansible-playbook -vv /usr/share/dci-openstack-agent/dci-openstack-agent.yml -e @/etc/dci-openstack-agent/settings.yml -e dci_topic=OSP10
ExecStart=-/usr/bin/ansible-playbook -vv /usr/share/dci-openstack-agent/dci-openstack-agent.yml -e @/etc/dci-openstack-agent/settings.yml -e dci_topic=OSP11
ExecStart=-/usr/bin/ansible-playbook -vv /usr/share/dci-openstack-agent/dci-openstack-agent.yml -e @/etc/dci-openstack-agent/settings.yml -e dci_topic=OSP12
SuccessExitStatus=0
User=dci-openstack-agent

[Install]
WantedBy=default.target
```

In this example, we do a run of OSP10, OSP11 and OSP12 everytime we start the
 agent.

### How to manually run the hooks

You can prepare a minimal playbook like this one:

```console
# cat run_hook.yml
- hosts: localhost
  tasks:
    - include_tasks: /etc/dci-openstack-agent/hooks/success.yml
```

Then call it using the `ansible` command:

```console
# ansible-playbook run_hook.yml
```

The tasks will be run on the local machine and nothing will be sent to the DCI server.

## How to switch between different configuration for the same remoteci/topic

DCI gives you the ability to have several configuration for a remoteci/topic. This allow you
for instance to run half of your deployments with an option and the other half without it.

This feature is not available yet on the interface. You need to use the `dcictl` CLI tool.

First, identify the ID of your remoteci and the topic.
```console
$ dcictl topic-list
$ dcictl remoteci-list
```

With this information, you can now associate the `rconfiguration` with the remoteci.

```console
$ dcictl remoteci-attach-rconfiguration bb16865b-6f88-4488-be8a-f87a54142eb0 --name ceph-enable --topic_id 7f1bc54b-790b-40c8-8993-af7e3910e7ca --component_types '["puddle_osp"]' --data "{}"
$ dcictl remoteci-attach-rconfiguration bb16865b-6f88-4488-be8a-f87a54142eb0 --name ceph-disabled --topic_id 7f1bc54b-790b-40c8-8993-af7e3910e7ca --component_types '["puddle_osp"]' --data "{}"
```

You can use `dcictl remoteci-list-rconfigurations` to validate your configuration:

```console
dcictl remoteci-list-rconfigurations cf7262db-b91a-450b-b6dc-5877e3753272                                                                                                         1331ms  Wed 12 Sep 2018 11:20:45 AM PDT
+--------------------------------------|--------------------|--------|-----------------|--------------------------------------+
|                  id                  |        name        | state  | component_types |               topic_id               |
+--------------------------------------|--------------------|--------|-----------------|--------------------------------------+
| bb16865b-6f88-4488-be8a-f87a54142eb0 |        ceph-enable | active | [u'puddle_osp'] | 7f1bc54b-790b-40c8-8993-af7e3910e7ca |
| bb16865b-6f88-4488-be8a-f87a54142eb0 |      ceph-disabled | active | [u'puddle_osp'] | 7f1bc54b-790b-40c8-8993-af7e3910e7ca |
+--------------------------------------|--------------------|--------|-----------------|--------------------------------------+
```


The name of the rconfiguration will be exposed in your hooks through
the `job_info.job.rconfiguration.name` variable. This is
an example where we trigger the OverCloud deployment with an exatra
parameter depending on the rconfiguration name.

```yaml
- name: Deploy OverCloud
  shell: "ansible-playbook -i inventory playbooks/prepare_overcloud.yml"
  args:
    chdir: /var/lib/dci-openstack-agent/ansible
  when: job_info.job.rconfiguration.name == 'no_ceph'

- name: Deploy OverCloud
  shell: "ansible-playbook -i inventory playbooks/prepare_overcloud.yml -e use_ceph=True
  args:
    chdir: /var/lib/dci-openstack-agent/ansible
  when: job_info.job.rconfiguration.name == 'with_ceph'
```
