# EDPM adoption

## Prerequisites

* Previous Adoption steps completed.

## Variables

(There are no shell variables necessary currently.)

## Pre-checks

## Procedure - EDPM adoption

* Create a [ssh authentication secret](https://kubernetes.io/docs/concepts/configuration/secret/#ssh-authentication-secrets) for the EDPM nodes:

  ```
  oc apply -f - <<EOF
    apiVersion: v1
    kind: Secret
    metadata:
        name: dataplane-adoption-secret
        namespace: openstack
    data:
        ssh-privatekey: |
    $(cat {{ edpm_privatekey_path }} | base64 | sed 's/^/        /')
    EOF
  ```

* Stop the nova services.

```bash

# Update the services list to be stopped

ServicesToStop=("tripleo_nova_api_cron.service"
                "tripleo_nova_api.service"
                "tripleo_nova_compute.service"
                "tripleo_nova_conductor.service"
                "tripleo_nova_libvirt.target"
                "tripleo_nova_metadata.service"
                "tripleo_nova_migration_target.service"
                "tripleo_nova_scheduler.service"
                "tripleo_nova_virtlogd_wrapper.service"
                "tripleo_nova_virtnodedevd.service"
                "tripleo_nova_virtproxyd.service"
                "tripleo_nova_virtqemud.service"
                "tripleo_nova_virtsecretd.service"
                "tripleo_nova_virtstoraged.service"
                "tripleo_nova_vnc_proxy.service")

echo "Stopping nova services"

for service in ${ServicesToStop[*]}; do
    echo "Stopping the $service in each controller node"
    $CONTROLLER1_SSH sudo systemctl stop $service
    $CONTROLLER2_SSH sudo systemctl stop $service
    $CONTROLLER3_SSH sudo systemctl stop $service
done
```

* Deploy OpenStackDataPlane:

  ```
  oc apply -f - <<EOF
    apiVersion: dataplane.openstack.org/v1beta1
    kind: OpenStackDataPlane
    metadata:
      name: openstack
      namespace: openstack
    spec:
      deployStrategy:
        deploy: true
      nodes:
        standalone:
          ansibleHost: {{ edpm_node_ip }}
          deployStrategy:
            deploy: false
          hostName: standalone
          node:
            ansibleSSHPrivateKeySecret: dataplane-adoption-secret
            ansibleVars: |
              ctlplane_ip: {{ edpm_node_ip }}
              internal_api_ip: 172.17.0.100
              storage_ip: 172.18.0.100
              tenant_ip: 172.19.0.100
              fqdn_internal_api: '{{'{{ ansible_fqdn }}'}}'
          role: edpmadoption
      roles:
        edpmadoption:
          deployStrategy:
            deploy: false
          networkAttachments:
            - ctlplane
            - internalapi
            - storage
            - tenant
          preProvisioned: true
          services:
            - configure-network
            - validate-network
            - install-os
            - configure-os
            - run-os
          env:
            - name: ANSIBLE_FORCE_COLOR
              value: "True"
            - name: ANSIBLE_ENABLE_TASK_DEBUGGER
              value: "True"
          nodeTemplate:
            # Defining the novaTemplate here means the nodes in this role are
            # computes and they will have Ceph RBD config overrides
            nova:
              cellName: cell1
              customServiceConfig: |
                [libvirt]
                images_type=rbd
                images_rbd_pool=vms
                images_rbd_ceph_conf=/etc/ceph/ceph.conf
                images_rbd_glance_store_name=default_backend
                images_rbd_glance_copy_poll_interval=15
                images_rbd_glance_copy_timeout=600
                rbd_user=openstack
                rbd_secret_uuid=$(oc get secret ceph-conf-files -o json | jq -r '.data."ceph.conf"' | base64 -d | grep fsid | sed -e 's/fsid = //')
              deploy: true
              novaInstance: nova
            # The Ansible Pod will mount these files and copy them to nodes in role
            extraMounts:
            - extraVolType: Ceph
              mounts:
              - mountPath: /etc/ceph
                name: ceph
                readOnly: true
              volumes:
              - name: ceph
                secret:
                  secretName: ceph-conf-files
            managementNetwork: ctlplane
            ansiblePort: 22
            ansibleSSHPrivateKeySecret: dataplane-adoption-secret
            ansibleUser: root
            ansibleVars: |
              service_net_map:
                nova_api_network: internal_api
                nova_libvirt_network: internal_api

              # edpm_network_config
              # Default nic config template for a EDPM compute node
              # These vars are edpm_network_config role vars
              edpm_network_config_template: templates/single_nic_vlans/single_nic_vlans.j2
              edpm_network_config_hide_sensitive_logs: false
              #
              # These vars are for the network config templates themselves and are
              # considered EDPM network defaults.
              edpm_network_config_bridge_name: br-ctlplane
              edpm_network_config_interface_name: eth1
              ctlplane_mtu: 1500
              ctlplane_subnet_cidr: 24
              ctlplane_gateway_ip: 192.168.122.1
              ctlplane_host_routes:
              - ip_netmask: 0.0.0.0/0
                next_hop: 192.168.122.1
              external_mtu: 1500
              external_vlan_id: 44
              external_cidr: '24'
              external_host_routes: []
              internal_api_mtu: 1500
              internal_api_vlan_id: 20
              internal_api_cidr: '24'
              internal_api_host_routes: []
              storage_mtu: 1500
              storage_vlan_id: 21
              storage_cidr: '24'
              storage_host_routes: []
              tenant_mtu: 1500
              tenant_vlan_id: 22
              tenant_cidr: '24'
              tenant_host_routes: []
              role_networks:
              - InternalApi
              - Storage
              - Tenant
              networks_lower:
                External: external
                InternalApi: internal_api
                Storage: storage
                Tenant: tenant

              # edpm_nodes_validation
              edpm_nodes_validation_validate_controllers_icmp: false
              edpm_nodes_validation_validate_gateway_icmp: false

              edpm_ovn_metadata_agent_DEFAULT_transport_url: $(oc get secret rabbitmq-transport-url-neutron-neutron-transport -o json | jq -r .data.transport_url | base64 -d)
              edpm_ovn_metadata_agent_metadata_agent_ovn_ovn_sb_connection: $(oc get ovndbcluster ovndbcluster-sb -o json | jq -r .status.dbAddress)
              edpm_ovn_metadata_agent_metadata_agent_DEFAULT_nova_metadata_host: $(oc get svc nova-metadata-internal -o json |jq -r '.status.loadBalancer.ingress[0].ip')
              edpm_ovn_metadata_agent_metadata_agent_DEFAULT_metadata_proxy_shared_secret: 1234567842
              edpm_ovn_metadata_agent_DEFAULT_bind_host: 127.0.0.1
              edpm_chrony_ntp_servers:
              - clock.redhat.com
              - clock2.redhat.com

              ctlplane_dns_nameservers:
              - $(oc get svc -l service=dnsmasq -o json | jq -r '.items[0].status.loadBalancer.ingress[0].ip')
              dns_search_domains: []
              edpm_ovn_dbs:
              - $(oc get ovndbcluster ovndbcluster-sb -o json | jq -r '.status.networkAttachments."openstack/internalapi"[0]')

              edpm_ovn_controller_agent_image: quay.io/podified-antelope-centos9/openstack-ovn-controller:current-podified
              edpm_iscsid_image: quay.io/podified-antelope-centos9/openstack-iscsid:current-podified
              edpm_logrotate_crond_image: quay.io/podified-antelope-centos9/openstack-cron:current-podified
              edpm_nova_compute_container_image: quay.io/podified-antelope-centos9/openstack-nova-compute:current-podified
              edpm_nova_libvirt_container_image: quay.io/podified-antelope-centos9/openstack-nova-libvirt:current-podified
              edpm_ovn_metadata_agent_image: quay.io/podified-antelope-centos9/openstack-neutron-metadata-agent-ovn:current-podified

              gather_facts: false
              enable_debug: false
              # edpm firewall, change the allowed CIDR if needed
              edpm_sshd_configure_firewall: true
              edpm_sshd_allowed_ranges: ['192.168.122.0/24']
              # SELinux module
              edpm_selinux_mode: permissive
              edpm_hosts_entries_undercloud_hosts_entries: []
              # edpm_hosts_entries role
              edpm_hosts_entries_extra_hosts_entries:
              - 172.17.0.80 glance-internal.openstack.svc neutron-internal.openstack.svc cinder-internal.openstack.svc nova-internal.openstack.svc placement-internal.openstack.svc keystone-internal.openstack.svc
              - 172.17.0.85 rabbitmq.openstack.svc
              - 172.17.0.86 rabbitmq-cell1.openstack.svc
              edpm_hosts_entries_vip_hosts_entries: []
              hosts_entries: []
              hosts_entry: []
    EOF
  ```
Note: Role vars will be inherited by nodes, more details [here](https://openstack-k8s-operators.github.io/dataplane-operator/inheritance/)

## Post-checks

* See that ansible jobs are running:

  ```
  while true; do oc logs -f `oc get pods | grep dataplane-deployment | grep Running| cut -d ' ' -f1` 2>/dev/null || echo -n .; sleep 1; done
  ```
