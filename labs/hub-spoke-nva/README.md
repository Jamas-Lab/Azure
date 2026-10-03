# Hub-and-spoke with an NVA and a cross-region spoke

**Exam area:** AZ-104, Implement and Manage Virtual Networking
**Run on:** 1 Oct 2026, UK South and West Europe
**Status:** Stages 0 to 5 run and recorded. Stages 6 to 11 designed but **not run**. Lab resources deleted after the session.

## Goal

Build a hub VNet with a network virtual appliance (NVA) and three spoke VNets, peer each spoke to the hub only, and find out step by step what it takes for one spoke to reach another through the hub. One spoke sits in a different region, so its link is a global VNet peering.

## Topology

| Role | Region | VNet | Address space | Subnet | VM | Private IP |
|---|---|---|---|---|---|---|
| Hub | UK South | JamaLabs-hub | 10.10.0.0/16 | snet-hub 10.10.0.0/24 | JamaLabs-uks-VM04 (the NVA) | 10.10.0.4 |
| Spoke 1 | UK South | JamaLabs-spoke1 | 10.20.0.0/16 | snet-spoke1 10.20.0.0/24 | JamaLabs-uks-VM01 | 10.20.0.4 |
| Spoke 2 | UK South | JamaLabs-spoke2 | 10.30.0.0/16 | snet-spoke2 10.30.0.0/24 | JamaLabs-uks-VM02 | 10.30.0.4 |
| Spoke 3 | West Europe | JamaLabs-weu-spoke3 | 10.40.0.0/16 | snet-spoke3 10.40.0.0/24 | JamaLabs-weu-VM03 | 10.40.0.4 |

- Four non-overlapping /16 ranges, so every peering is allowed.
- VNets in one resource group, VMs, NICs and disks in another. Both are long-lived groups that hold other resources, so cleanup is done resource by resource.
- No public IPs on any VM.
- Regular (non-Spot) VMs, size Standard_D2as_v4, because the Spot quota in UK South was full.

## Peerings

Six links, one per direction for each spoke. Spokes are peered only to the hub, never to each other: a direct spoke-to-spoke link would bypass the NVA.

| Link | Type |
|---|---|
| hub-to-spoke1, spoke1-to-hub | Regional |
| hub-to-spoke2, spoke2-to-hub | Regional |
| hub-to-spoke3, spoke3-to-hub | Global |

Settings on all six: virtual network access Yes, forwarded traffic Yes, gateway transit No, use remote gateways No (no gateways exist in this lab). The hub-side link of each pair was seen as Connected and Fully Synchronized; the spoke-side links are inferred from that, not read directly.

## Routing design

One route table per spoke, two routes each, all with next hop type VirtualAppliance and the NVA (10.10.0.4) as the next hop IP. Each spoke sends traffic for the other two spoke ranges to the NVA. A route table does nothing until it is associated with a subnet. Only `rt-spoke1` was built and attached in this run (Stage 4).

## What was run and what happened

| Stage | Change | Expected | Result |
|---|---|---|---|
| 0 | Created 4 VNets, 4 subnets and 4 VMs. NIC IP forwarding off on the NVA. | Four running VMs at the .4 addresses | Confirmed. Four VMs running, private IPs as in the table. |
| 1 | Added the six peerings. No route tables. | NVA reachable from every spoke. No spoke reaches another, because peering is not transitive. | Spoke 2 to spoke 3: 0 of 137 replies, no reply and no error. From the NVA, all three spokes replied. Spoke 2 to spoke 1 showed no replies when captured. Spoke-to-NVA direction was not recorded. |
| 2 | Compared round-trip times from the Stage 1 pings. | Cross-region higher than same-region | From the NVA: spoke 1 1.65 ms (7 pings), spoke 2 0.79 ms (3 pings), spoke 3 in West Europe 7.85 ms (4 pings) and 7.50 ms on a second run. About 6 to 7 ms slower cross-region. Short runs, not a benchmark. |
| 3 | Read the effective routes on the NVA's NIC. | System routes to all three spoke ranges | Confirmed with the CLI: 10.10.0.0/16 VnetLocal, 10.20.0.0/16 and 10.30.0.0/16 VNetPeering, 10.40.0.0/16 VNetGlobalPeering. All Default and Active, no user routes. The portal view did not show the 10.40 row; the CLI did. Next hop labels differ between tools. |
| 4 | Added and attached `rt-spoke1` (to-spoke2 and to-spoke3 via 10.10.0.4). NIC forwarding still off. | Ping from spoke 1 to spoke 2 fails | Ping failed. Network Watcher next hop returned VirtualAppliance 10.10.0.4 via `rt-spoke1`, so the route was being used. Effective-route rows on the spoke 1 VM were not captured. |
| 5 | Turned IP forwarding on in Azure for the NVA's NIC. | Still fails, because the NVA's operating system does not forward | NIC IP forwarding set to true. The ping test and the in-OS forwarding value were not run; the lab stopped here. |

## Designed but not run

These stages are reasoning, not results.

| Stage | Change | Reasoning |
|---|---|---|
| 6 | Enable forwarding inside the NVA's operating system | The request reaches spoke 2, but spoke 2 has no route back to 10.20.0.0/16, so the reply is dropped |
| 7 | Add `rt-spoke2` | Spoke 1 and spoke 2 work both ways. Spoke 1 to spoke 3 still fails for the same missing-return-route reason |
| 8 | Add `rt-spoke3` | All pairs work through the NVA. Pairs involving spoke 3 are slower, because traffic crosses to UK South and back |
| 9 | Read effective routes on spoke NICs | User routes Active, system 10.0.0.0/8 to None stays Active (different, less specific prefix) |
| 10 | Turn Allow forwarded traffic off, one link at a time | Unknown. The point is to record which links break the ping |
| 11 | Replace the two /16 routes with one 10.0.0.0/8 route per spoke | The system 10.0.0.0/8 route goes Invalid, because it has the same prefix and the user route wins |

## What this lab showed

- **Peering is not transitive.** Two spokes peered to the same hub cannot reach each other (0 of 137 replies).
- **Effective routes can differ by tool.** The CLI showed a route the portal view did not.
- **A route to an NVA is not enough on its own.** With `rt-spoke1` attached and the next hop confirmed as the NVA, the ping still failed. Why it failed is the lab's open question: NIC forwarding was off at that point, and the OS-level and return-route checks were not run.
- **Cross-region peering added about 6 to 7 ms** on short runs.

## Cleanup

Order used: detach each route table from its subnet, delete the route tables, delete the VMs, delete leftover NICs and disks, delete the six peering links, delete the four VNets. Both resource groups were kept.

## Next

Resume at Stage 5: ping from spoke 1 to spoke 2, then check `/proc/sys/net/ipv4/ip_forward` on the NVA, and carry on through Stage 11.
