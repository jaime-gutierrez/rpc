# Configure Neutron Layer 3 Provider & Tenant Networks with with Floating IPs

###Example provider gateway and tenant inside networks
````
NETWORK         VLAN_ID   NETWORK       TAGGED INTERFACE
RPC_GATEWAY_NET 239       10.239.0.0/22     (bond1)  (A PROVIDER NETWORK) (INTENET ACCESS)
RPC_INSIDE_NET  242       10.242.0.0/22     (bond1)  (A TENANT NETWORK)   (NO INTERNET ACCESS)
````

### Set Variables for Provider and Tenant Networks
### This scenario will setup the networks for the the 'admin' tenant

````
TENANT_ID=$(openstack project list | awk '/admin/{print $2}')
GW_PROVIDER_VLAN_ID=239
GW_PROVIDER_CIDR="10.239.0.0/22"
GW_PROVIDER_GATEWAY="10.239.0.1"
GW_PROV_POOL_START_IP="10.239.0.10"
GW_PROV_POOL_END_IP="10.239.3.254"
INSIDE_TENANT_VLAN_ID=242
INSIDE_TENANT_CIDR="10.242.0.0/22"
INSIDE_NET_GW="10.242.0.1"
DNS1=${INSIDE_NET_GW}
DNS2="8.8.8.8"
````

### Run the following

````
neutron net-create --provider:physical_network=vlan --provider:network_type=vlan --provider:segmentation_id=${GW_PROVIDER_VLAN_ID} --router:external GATEWAY_NET
GW_NET_ID=$(neutron net-list | awk '/GATEWAY/{print $2}')
neutron net-create --provider:physical_network=vlan --provider:network_type=vlan --provider:segmentation_id=${INSIDE_TENANT_VLAN_ID} --tenant-id ${TENANT_ID} INSIDE_NET
INSIDE_NET_ID=$(neutron net-list | awk '/INSIDE/{print $2}')
neutron subnet-create GATEWAY_NET ${GW_PROVIDER_CIDR} --name GATEWAY_SUBNET --gateway=${GW_PROVIDER_GATEWAY} --allocation-pool start=${GW_PROV_POOL_START_IP},end=${GW_PROV_POOL_END_IP}  --enable_dhcp false
neutron subnet-create INSIDE_NET ${INSIDE_TENANT_CIDR} --name INSIDE_SUBNET --dns-nameservers list=true ${DNS1} ${DNS2}
INSIDE_SUBNET_ID=$(neutron subnet-list | awk '/INSIDE_SUBNET/{print $2}')
neutron router-create RTR-PUBLIC --ha False
ROUTER_ID=$(neutron router-list | awk '/RTR-PUBLIC/{print $2}')
neutron router-gateway-set --enable-snat ${ROUTER_ID} ${GW_NET_ID}
neutron router-interface-add ${ROUTER_ID} ${INSIDE_SUBNET_ID}
````

### Create a test instance and floating ip
#### Set INSTANCE_NUM only once
````
INSTANCE_NUM=1
````
#### If you decide to add another instance after running everything, start from this point.

````
IMAGE=$(glance image-list | awk '/Trusty/{print $2}')
FLAVOR=2
SECURITY_GROUP=rpc-support
AVAILABILITY_ZONE=nova
SSHKEY=rpc_support
NETWORK_UUID=${INSIDE_NET_ID}

FLOATING_IP_ID=$(neutron floatingip-create --tenant-id ${TENANT_ID} ${GW_NET_ID} | awk  '/ id /{print $4}')
INSTANCE_NAME=$"test-rax-${INSTANCE_NUM}-${COMPUTE_NODE}"
COMPUTE_NODE=server1
nova boot \
    --image ${IMAGE} \
    --flavor ${FLAVOR} \
    --security-groups ${SECURITY_GROUP} \
    --availability-zone ${AVAILABILITY_ZONE}:${COMPUTE_NODE} \
    --key-name ${SSHKEY} \
    --nic net-id=${NETWORK_UUID} \
    ${INSTANCE_NAME}
((INSTANCE_NUM++));
sleep 10;
nova list --tenant ${TENANT_ID} | grep ${INSTANCE_NAME} | awk '/INSIDE_NET/{print $14}' | sed 's/INSIDE_NET=//'
INSTANCE_INSIDE_NET_IP=$(nova list --tenant ${TENANT_ID} | grep ${INSTANCE_NAME} | awk '/INSIDE_NET/{print $14}' | sed 's/INSIDE_NET=//')
INSTANCE_INSIDE_NET_PORT_ID=$(neutron port-list | grep ${INSTANCE_INSIDE_NET_IP} | awk '{print $2}')
neutron floatingip-associate ${FLOATING_IP_ID} ${INSTANCE_INSIDE_NET_PORT_ID}

````


