<h1>Network Security Defense Framework: ACLs, Port Security, DHCP Snooping, and Dynamic ARP Inspection</h1>

<p>
- The goal of this project is to design and deploy a secure, hierarchical enterprise network that models a real-world corporate environment using an Access-Distribution-Core architecture. The network is divided into three segmented trust zones: Executives, Operations, and a restricted Lobby guest network, with each VLAN being under the principle of least privilege to limit unnecessary lateral movement and reduce attack exposure. At the core of the design is a Layer 3 distribution switch handling inter-VLAN routing and acting as a key security enforcement boundary between internal user networks and external server resources.
To secure the infrastructure, multiple layered security controls are implemented across the network, including VLAN segmentation, DHCP services with centralized control, Spanning Tree optimization, and inter-VLAN routing via router-on-a-stick. On top of the base connectivity design, additional security mechanisms are deployed to enforce defense-in-depth protection, including Port Security for MAC address binding at the access layer, DHCP Snooping to prevent rogue DHCP servers and build trusted IP-to-MAC bindings, Dynamic ARP Inspection (DAI) to prevent ARP spoofing by validating traffic against the DHCP Snooping Binding Table, and Access Control Lists (standard and extended) to enforce traffic filtering between VLANs and control access to critical server resources. These combined measures ensure that both Layer 2 and Layer 3 attack vectors are actively monitored, restricted, and mitigated throughout the network.
The image below shows the topology before the configuration. At the center of this design is a Layer 3 Switch acting as the distribution layer, which handles high-speed Inter-VLAN routing and serves as the security boundary for the internal subnets. A routed transit link between the L3 Switch and R1 using the 10.0.0.0/30 subnet will be used to ensure a clear separation between the internal switching fabric and the edge routing. I have labeled the subnets and interfaces because these represent our Security Enforcement Points. This labeling is important because security measures are interface-dependent; an ACL applied to the wrong port or a trust boundary set on the wrong link can either leave the network wide open or shut down legitimate business operations.
</p>
<p>
<img width="1215" height="524" alt="image" src="https://github.com/user-attachments/assets/f40f03d1-c8f9-49fb-a8ee-d3eaecfca61c" />
</p>
<br>

<p>
- 
</p>
<p>

</p>
<br>

<p>
- 
</p>
<p>

</p>
<br>

<p>
- 
</p>
<p>

</p>
<br>

<p>
- 
</p>
<p>

</p>
<br>

<p>
- 
</p>
<p>

</p>
<br>

<p>
- 
</p>
<p>

</p>
<br>

<p>
- 
</p>
<p>

</p>
<br>


