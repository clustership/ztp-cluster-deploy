# Assisted Installer operator (ipv6+disconnected) #
This is a set of playbooks to automate the deployment of Assisted Installer Operator
in a disconnected way, assuming ipv6 networking.

## Requirements ##

1. Podman.
2. Ansible.
3. Ansible modules `containers.podman` and `community.general`. You can install all the requirements with: 

  ```bash
  ansible-galaxy collection install -r requirements.yaml
  ```
        
## Steps ##

1. git clone the `ztp-cluster-deploy` repo
2. go to `install-ai-operator` dir
3. You need make a copy `inventory/hosts.sample` file and name it `hosts` under same directory.
4. Modify the [all:vars] section at `inventory/hosts` file based on your env.
5. If you want that the playbook creates the initial provisioner cluster for you, please run the
create-provisioner-cluster tag. It will generate a cluster, then you can copy the kubeconfig file
from /root/ocp/auth/kubeconfig installer vm into your local provisioner host. You can set that
on the inventory, for next steps.
6. Start the deployment with `prepare-environment` tag:

      ```console
      $ sudo ansible-playbook playbook.yml -vvv -i inventory/hosts --tags=prepare-environment
      ```

7. Create the OpenShift disconnected mirror if needed, by running the create-olm-mirror and
mirror-ai-images tags.

      ```console
      $ sudo ansible-playbook playbook.yml -vvv -i inventory/hosts --tags=offline-mirror-olm
      ```

8. Mirror Assisted Installer images. It will just use the default mirror settings, or it can also mirror
into an existing one, by providing the `provisioner_cluster_registry` var. It will also create a local
http server, where it will copy the RHCOS images. By default it will use the same VM used for the
disconnected registry, but you can provide your own using the `disconnected_http_server` var:

      ```console
      $ sudo ansible-playbook playbook.yml -vvv -i inventory/hosts --tags=mirror-ai-images
      ```
    
9. Finally install the Assisted Installer operator. You need to set the
`provisioner_cluster_kubeconfig` var in inventory:

      ```console
      $ sudo ansible-playbook playbook.yml -vvv -i inventory/hosts --tags=install-assisted-installer
      ```

10. Once deployed, you could create clusters using CRDs. This is going to be covered on a different section.

## Deploy with static IPs ##

in order to deploy the cluster, it is mandatory to have a DHCP server and DNS resolution at switch level.
However, it is possible to deploy with SLAAC IPv6 addresses with some workaround:

1. Create a local DNS server (using coredns, dnsmasq, etc...). It needs to contain entries for
the disconnected registry (if used), and api / apps endpoint. A sample dnsmasq.conf follows:

      ```
      strict-order
      bogus-priv
      bind-interfaces
      listen-address=2620:52:0:1310:ce29:8e60:eb92:e5f6
      except-interface=lo
      # disable dhcp service, we just need dns as we have slaac
      no-dhcp-interface=baremetal
      domain=clus2.t5g.lab.eng.bos.redhat.com

      # static host-records
      address=/apps.test-operator.clus2.t5g.lab.eng.bos.redhat.com/2620:52:0:1310::fb
      host-record=api.test-operator.clus2.t5g.lab.eng.bos.redhat.com,2620:52:0:1310::fa
      host-record=test-operator-installer.clus2.t5g.lab.eng.bos.redhat.com,2620:52:0:1310::e8
      ```
2. Be sure to inject the fixed static ip to your mirror, so it is resolved to right one.
Mirror may normally be located on an auxiliary virtual machine. In order to setup static
IP, you can simply run:

      ```
      nmcli connection modify 'Wired Connection' ipv6.address 2006:ac81::1105/64
      nmcli connection modify 'Wired Connection' ipv6.method manual
      nmcli connection up 'Wired Connection'
      nmcli connection reload
      ```

3. On your bootstrap machine, be sure to inject the right nameserver, so it can resolve your
definitions. Also if you go with ipv6, be sure to disable any possible ipv4 dhcp entries:

      ```
      nmcli connection mod 'Wired connection 1' ipv6.dns "2620:52:0:1310:ce29:8e60:eb92:e5f6" ipv6.dns-search "clus2.t5g.lab.eng.bos.redhat.com" ipv6.dns-priority 50 ipv4.ignore-auto-dns yes connection.autoconnect yes
      nmcli conn up 'Wired connection 1'
      ```

4. Once master nodes boot, do the same injection for those. You can do it by injecting some
service that is triggered after NetworkManager boots. You will need to add it on your primary
NIC and also in the physical interface for the OVS bridge. Please also ensure that the
hostname is set properly:

      ```
      nmcli connection mod 'Wired Connection' ipv6.dns "2620:52:0:1310:ce29:8e60:eb92:e5f6" ipv6.dns-search "clus2.t5g.lab.eng.bos.redhat.com" ipv4.ignore-auto-dns yes ipv6.dns-priority 50
      nmcli connection mod 'Wired connection 1' ipv6.dns "2620:52:0:1310:ce29:8e60:eb92:e5f6" ipv6.dns-search "clus2.t5g.lab.eng.bos.redhat.com" ipv4.ignore-auto-dns yes ipv6.dns-priority 50
      nmcli connection mod 'ovs-if-br-ex' ipv6.dns "2620:52:0:1310:ce29:8e60:eb92:e5f6" ipv6.dns-search "clus2.t5g.lab.eng.bos.redhat.com" ipv4.ignore-auto-dns yes ipv6.dns-priority 50
      systemctl restart NetworkManager
      systemctl set-hostname <cluster_name>-master-<number>-<domain>
      ```
5. With those modifications, your cluster will be properly resolving the api/apps endpoints, thanks
to your custom name server, and there is no need to have an integrated dhcp6/dns service in the lab.
