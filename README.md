# F5 Ramp node setup

A ramp node is needed to fully integrate the F5 load balancer with an OpenShift cluster that runs its networking using openshift-sdn. OpenShiftSDN being an overlay network using VxLAN, the ramp node becomes a logical gateway through which traffic is moderated to and from the F5 appliance. The lynchpin of the setup is the ipip tunnel that is created between F5 and the ramp-node.

OpenShift documentation describes the steps required to setup the ramp node [here](https://docs.openshift.org/latest/install_config/routing_from_edge_lb.html#establishing-a-tunnel-using-a-ramp-node). But one of the key issues is persistence of the setup i.e. the ipip tunnel, the Openvswitch rules etc do not survive a reboot of the machine. This article is about putting together the steps into a nifty script that one can just setup in an init script e.g. a systemd unit.

```
#!/bin/bash
set -e
#
# Usage: setup-ramp-node-sdn <F5-IP>
# where: <F5-IP> is the IP address of the internal interface
# of the F5 router.
#
# Examples:
# ./setup-ramp-node-sdn 10.3.89.191
# where: 10.3.89.235 is the IP address of the F5 router.


# Recreate the IP-IP tunnel from the ramp node to the F5.
# Usage: _recreate_tunnel_to_f5 <f5-ip> <f5-tunnel-ip> <ramp-tunnel-ip>
#
# Examples:
# _recreate_tunnel_to_f5 10.3.89.191 10.3.91.216 10.3.91.217
#
function _recreate_tunnel_to_f5() {
  local f5ip=$1
  local f5_tunnel_ip=$2
  local rampnode_tunnel_ip=$3
  local tun_device=tun1
  echo ""
  echo " - Deleting IPIP tunnel on ramp node ... "
  ip tunnel del ${tun_device} || true
  echo " - Creating IPIP tunnel on ramp node to f5 ip address ${f5ip} ..."
  ip tunnel add ${tun_device} mode ipip remote ${f5ip} dev eth0
  echo " - Adding local ramp node tunnel IP ${rampnode_tunnel_ip} ... "
  ip addr add ${rampnode_tunnel_ip} dev ${tun_device}
  echo " - Setting link state to up for tunnel ${tun_device} ... "
  ip link set ${tun_device} up
  echo " - Adding route to F5 tunnel ${f5_tunnel_ip} via ${tun_device} ..."
  ip route add ${f5_tunnel_ip} dev ${tun_device}
  echo " - Pinging F5 end of the tunnel ... "
  ping -c 5 ${f5_tunnel_ip}
  if [ $? -ne 0 ]; then
   echo "ERROR: Unable to ping F5 tunnel IP ${f5_tunnel_ip} - exiting!"
   exit 1
  fi
  echo " - IPIP tunnel setup done."
  echo ""
} # End of function _recreate_tunnel_to_f5.

#
# Cleanup OVS rules by restarting the OpenShift SDN service and getting to
# a consistent/clean state.
# Usage: _cleanup_osdn_ovs_rules <service-name> <max-retries>
#
# Examples:
# _cleanup_osdn_ovs_rules origin-node 99
#
function _cleanup_osdn_ovs_rules() {
  local svcname=${1:-"openshift-node"}
  local max_retries=${2:-42}
  echo " - Cleaning up OVS rules ... "
  echo " - Bringing down OpenShift SDN node ${svcname} ... "
  systemctl stop ${svcname}
  echo " - Setting link state to lbr0 to down ... "
  ip link set lbr0 down ||:
  echo " - Deleting bridge lbr0 ... "
  brctl delbr lbr0 ||:
  echo " - Deleting OVS bridge br0 ... "
  ovs-vsctl del-br br0 ||:
  echo " - Restarting OpenShift SDN ... "
  systemctl start ${svcname}
  echo " - Waiting for OpenShift SDN service to start up ..."
  retries=0
  while ! ovs-ofctl -O OpenFlow13 dump-flows br0 | grep "table=0"; do
    retries=$((retries + 1))
    if [ $retries -ge ${max_retries} ]; then
      echo ""
      echo "ERROR: Max retries $retries exceeded. Exiting ..."
      exit 1
    else
      echo -n "."
      sleep 1
    fi
  done
  echo ""
} # End of function _cleanup_osdn_ovs_rules.

#
# main():
#
if [ $# -lt 1 ]; then
  echo "Usage: $0 <f5-internal-ip>"
  echo " where: <f5-internal-ip> is the IP address of the internal"
  echo " interface of the F5 router."
  exit 1
fi
#
# If your environment settings are customized, you can either set the env
# variables to the customized values or else edit this script.
#
F5_TUNNEL_IP=${OPENSHIFT_F5_TUNNEL_IP:-"10.3.91.216"}
RAMP_TUNNEL_IP=${OPENSHIFT_RAMP_TUNNEL_IP:-"10.3.91.217"}
CLUSTER_NETWORK=${OPENSHIFT_CLUSTER_NETWORK:-"10.128.0.0/14"}
OPENSHIFT_NODE_SERVICE=${OPENSHIFT_NODE_SERVICE:-"openshift-node"}
MAX_RETRIES=${OPENSHIFT_SDN_MAX_RETRIES:-42}
SDN_TAP1_ADDR=${OPENSHIFT_SDN_TAP1_ADDR}

_recreate_tunnel_to_f5 "$1" "${F5_TUNNEL_IP}" "${RAMP_TUNNEL_IP}"

_cleanup_osdn_ovs_rules "${OPENSHIFT_NODE_SERVICE}" "${MAX_RETRIES}"

echo ""
echo " - Setting up OpenShift SDN for F5 config ... "
tap1=$(ip -o -4 addr list tun0 | awk '{print $4}' | cut -d/ -f1 | head -n 1)
subaddr=$(echo ${SDN_TAP1_ADDR:-"${tap1}"} | cut -d "." -f 1,2,3)
export RAMP_SDN_IP=${subaddr}.254
echo " - RAMP_SDN_IP set to ${RAMP_SDN_IP}"
echo " - Adding IP address ${RAMP_SDN_IP} to tun0 ... "
ip addr add ${RAMP_SDN_IP} dev tun0
ipflowopts="cookie=0x999,ip"
arpflowopts="cookie=0x999, table=0, arp"

echo " - Modifying OVS rules for SNAT ... "
echo " - Adding OVS flow to map ${F5_TUNNEL_IP} to ${RAMP_SDN_IP} ... "
ovs-ofctl -O OpenFlow13 add-flow br0 \
"${ipflowopts},nw_src=${F5_TUNNEL_IP},actions=mod_nw_src:${RAMP_SDN_IP},resubmit(,0)"
echo " - Adding OVS flow to map ${RAMP_SDN_IP} to ${F5_TUNNEL_IP} ... "
ovs-ofctl -O OpenFlow13 add-flow br0 \
"${ipflowopts},nw_dst=${RAMP_SDN_IP},actions=mod_nw_dst:${F5_TUNNEL_IP},resubmit(,0)"
echo " - Adding OVS flow for arp to/from ${RAMP_SDN_IP} ... "
ovs-ofctl -O OpenFlow13 add-flow br0 \
"${arpflowopts}, arp_tpa=${RAMP_SDN_IP}, actions=output:2"
ovs-ofctl -O OpenFlow13 add-flow br0 \
"${arpflowopts}, priority=200, in_port=2, arp_spa=${RAMP_SDN_IP}, arp_tpa=${CLUSTER_NETWORK}, actions=goto_table:5"
ovs-ofctl -O OpenFlow13 add-flow br0 \
"arp, table=5, priority=300, arp_tpa=${RAMP_SDN_IP}, actions=output:2"
echo " - Adding OVS flow for traffic to ${RAMP_SDN_IP} ... "
ovs-ofctl -O OpenFlow13 add-flow br0 \
"ip,table=5,priority=300,nw_dst=${RAMP_SDN_IP},actions=output:2"
echo " - Adding OVS flow for traffic to tunnel ${F5_TUNNEL_IP} ... "
ovs-ofctl -O OpenFlow13 add-flow br0 \
"${ipflowopts},nw_dst=${F5_TUNNEL_IP},actions=output:2"

echo " - All done."
```

That may look like an overwhelming script, but there are three things that it does:

1. Recreate the ipip tunnel between the F5 node and the ramp node (the ramp node is where this script should be run)
2. Cleanup the OVS rules (by restarting the openshift-node process)
3. Add the extra OVS rules for managing the F5 traffic

One may be tempted to just put that script in a systemd unit file and be done with the headache of reboot survival forever, but some key variables need to be understood and set accordingly. Just a handful of them, really -

1. ${OPENSHIFT_F5_TUNNEL_IP} - This is the IP address of the tunnel device on the F5 device. Default value:- "10.3.91.216"
2. ${OPENSHIFT_RAMP_TUNNEL_IP} - This is the IP address chosen for the tunnel device on the ramp node. Default value:-"10.3.91.217". These addresses are just any valid L3 addresses that the underlay network will accept.
3. ${OPENSHIFT_CLUSTER_NETWORK} - This is the pod subnet that the cluster is operating with. i.e. all pods belong to this CIDR. Default value:-"10.128.0.0/14"
4. ${OPENSHIFT_NODE_SERVICE} - Name of the systemd service for the openshift node process. Default value :- "openshift-node"
5. ${OPENSHIFT_SDN_TAP1_ADDR} - IP address of the 'tun0' device on the ramp node as setup by openshift-sdn lease. There is no default value. If found unset, it will be picked up automatically by this script. Set it up only if you want to override it (usually not needed).
