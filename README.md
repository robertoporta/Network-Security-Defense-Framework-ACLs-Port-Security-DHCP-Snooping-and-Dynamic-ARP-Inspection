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
- As the first step, I configure the transit connection between R1 and the Layer 3 Switch. The serial connection between R1 and R2 has already been pre-configured, with R1 using S0/0/0 (203.0.113.1/30) and R2 using S0/0/0 (203.0.113.2/30). Additionally, R2 already has OSPF pre-configured to advertise the server farm networks. Because the Layer 3 Switch will be operating as a Layer 2 distribution switch for the transit connection, I implement a Router-on-a-Stick design on R1 using dedicated subinterfaces for each VLAN. This allows VLANs 10, 20, 30, and 40 to securely traverse the trunk link while still giving R1 Layer 3 visibility into the internal networks. I configured four subinterfaces on R1 tied to their respective VLANs: g0/0.10 for the Executives VLAN using IP address 100.1.1.254/24, g0/0.20 for the Operations VLAN using IP address 100.2.2.254/24, g0/0.30 for the Lobby VLAN using IP address 100.3.3.254/24, and g0/0.40 for the dedicated transit VLAN using IP address 10.0.0.1/30. This allows DHCP relay traffic and routed enterprise traffic to cross the trunk correctly while maintaining logical segmentation between departments. I then configure OSPF on R1 so it can dynamically advertise the internal enterprise networks while also learning routes to the protected servers from R2. This ensures full end-to-end routing between the client VLANs and the HTTPS/DNS servers without relying on static routes.
</p>
<p>
<img width="776" height="411" alt="image" src="https://github.com/user-attachments/assets/6d150b68-6fcd-42b5-9c3f-db97bb2fe017" />
</p>
<p>
<img width="768" height="405" alt="image" src="https://github.com/user-attachments/assets/892392fa-ec95-4732-b2ea-8145e10b56f5" />
</p>
<br>

<p>
- Next, I configure R1 as the central DHCP server for the entire organization to eliminate the overhead of manual IP management. I define three distinct pools for the Executives, Operations, and Lobby subnets. To maintain consistency across the enterprise network, I use the last usable IP address in each subnet as the default gateway. This is a common best practice because it creates a predictable gateway addressing scheme that simplifies administration and troubleshooting. To prevent IP conflicts with statically assigned devices like printers or management interfaces, I explicitly exclude the first ten addresses of each unique 100.x.x.x subnet. I also configure all clients to use the centralized DNS server at 192.168.2.100, which is part of the secure server side of the topology and will provide DNS resolution services throughout the lab environment. 
</p>
<p>
<img width="739" height="431" alt="image" src="https://github.com/user-attachments/assets/47e9ac40-9f3d-425c-b18c-295475dad66b" />
</p>
<br>

<p>
- In this step, I create the VLAN structure on the Layer 3 Switch and prepare it to operate as the central distribution device for the enterprise. I assigned the Executives to VLAN 10, Operations to VLAN 20, and the Lobby to VLAN 30, while VLAN 40 is used as the dedicated transit VLAN between the Distribution switch and R1. I configure interface G0/1 as a trunk port and explicitly allow VLANs 10, 20, 30, and 40 across the link so all departmental traffic and transit traffic can securely traverse the connection to R1.
  
In a professional network design, manually designating the distribution switch as the root bridge is a mandatory configuration step to ensure deterministic traffic flow and prevent the Spanning Tree Protocol (STP) pruning that occurs by default. Without this command, switches use a standard priority (32768) and elect a root bridge based on the lowest MAC address. If an access switch, such as one in the Lobby, happens to have that lowest address, it becomes the logical center of the network. The distribution switch then views its high-speed uplink to the router, as a side path that does not lead toward the root, triggering STP to logically block or prune VLANs on that trunk to eliminate potential loops. By manually lowering the STP priority to 4096 using the “spanning-tree vlan” command, I override the MAC address tie-breaker and force the distribution switch to become the heart of the star topology. This keeps all critical trunk ports in a forwarding state and ensures that important traffic, such as DHCP broadcasts, can always reach the router without interruption.
</p>
<p>
<img width="773" height="350" alt="image" src="https://github.com/user-attachments/assets/087c8e9f-3656-4bed-8c10-8dd31be09999" />
</p>
<br>

<p>
- To finish L3SW’s initial setup, I configure it to operate as a high-performance Layer 2 switch for the transit link to R1. It is technically and strategically sound to use a Layer 3 Switch even for Layer 2 tasks because these devices have significantly more processing power, higher switching fabric capacity, and larger TCAM tables than standard access switches. Positioned directly after the access layer in our model, the L3 switch acts as a powerful secondary filter for the core. By maintaining this hardware at the distribution layer, we ensure the network has the Layer 3 option available for future growth, allowing for a seamless transition to advanced routing or hardware-accelerated ACLs without requiring a hardware replacement. I created VLAN 40 as a dedicated transit VLAN between the L3 Switch and R1, as this VLAN logically separates the router transit traffic from the user VLANs while allowing all VLAN traffic to traverse the trunk link securely. I also configured interface G0/1 as a trunk to carry the necessary VLAN traffic to R1, ensuring the DHCP DORA process can reach the server through a secure, high-speed path. Additionally, for extra security I allowed only the required VLANs 10, 20, 30, and 40.
</p>
<p>
<img width="762" height="336" alt="image" src="https://github.com/user-attachments/assets/16bf8680-3060-4127-ac4a-f9bc9ed5ad6b" />
</p>
<br>

<p>
- Moving onto the 3 access switches, I configure them so client traffic can properly reach the Layer 3 switch SVIs and successfully obtain DHCP addresses from R1. Each access switch is dedicated to a single department: SW1 for Executives, SW2 for Operations, and SW3 for the Lobby. Interface F0/1 on each switch is used as the uplink to the Layer 3 Switch, while interfaces F0/2-24 are assigned to end-user devices. The configuration across the 3 access switches is nearly identical, as I configure the uplink as a trunk port and allow only the required VLANs for extra security, with the only difference being the assigning of the client-facing ports to the correct VLAN. This ensures that DHCP broadcasts from the PCs are carried across the trunk link to the correct SVI on the Layer 3 Switch.
</p>
<p>
<img width="774" height="384" alt="image" src="https://github.com/user-attachments/assets/31fe881a-8c87-42f5-9fba-51ff02486f3e" />
</p>
<p>
<img width="769" height="383" alt="image" src="https://github.com/user-attachments/assets/b7f2cc57-5a55-48c4-8ac2-21ff46f0e9bb" />
</p>
<p>
<img width="752" height="377" alt="image" src="https://github.com/user-attachments/assets/4673a7c4-d765-46cc-be85-43587bd8fe55" />
</p>
<br>

<p>
- With the relay and pools active, I move to the end-user devices to verify that the DHCP DORA process (Discover, Offer, Request, Acknowledge) is completing successfully across the Layer 3 boundary. By navigating to the IP Configuration tab of each workstation and selecting the "DHCP" option, I confirmed that each device received its unique 100.x.x.x lease. The images below show 1 PC for each subnet, this confirms that the transit link between R1 and the L3SW is routing properly and that the helper-address logic is functional before we begin layering on security protocols.
</p>
<p>
<img width="777" height="169" alt="image" src="https://github.com/user-attachments/assets/bee63651-960f-4532-9772-646b68fecd18" />
</p>
<p>
<img width="776" height="166" alt="image" src="https://github.com/user-attachments/assets/300d720f-a695-48f9-91a1-fde773c9b589" />
</p>
<p>
<img width="778" height="165" alt="image" src="https://github.com/user-attachments/assets/50e97faa-25c3-4358-9769-238415bd007c" />
</p>
<br>

<p>
- Now we can begin to implement the security measures. First, to prevent physical network breaches at the edge, I add Port Security on the 3 access switches (SW1, SW2, SW3). With the interfaces in the switches being access ports, I now apply the Port Security feature and related security parameters to those existing access interfaces. By using the "sticky" keyword, I allow the switch to dynamically learn the authorized MAC address of the connected PC and save it to the configuration. Although I don’t have to run the violation shutdown command because by default the shutdown mode is used, I still run the command as good practice. Cisco switches support three violation modes for Port Security: protect, restrict, and shutdown. Protect silently drops unauthorized traffic without logging, restrict drops unauthorized traffic while generating logs and incrementing counters, and shutdown places the port into an err-disabled state. I intentionally use the default shutdown mode because it provides the strongest security response by completely disabling the compromised interface the moment an unauthorized device is detected. If an intruder attempts to unplug a company laptop and connect a rogue device, the switch detects the unauthorized MAC and triggers a shutdown violation, physically disabling the port to prevent any further entry.
</p>
<p>
<img width="751" height="112" alt="image" src="https://github.com/user-attachments/assets/192faed6-07fb-4c4f-adab-d915cbe9bc03" />
</p>
<p>
<img width="776" height="118" alt="image" src="https://github.com/user-attachments/assets/0f722357-75be-4292-ab08-d0fbf66a64f8" />
</p>
<p>
<img width="773" height="117" alt="image" src="https://github.com/user-attachments/assets/457b6b74-d221-440b-810d-444034fb7af3" />
</p>
<br>

<p>
- Then I implement DHCP Snooping to protect the infrastructure from rogue DHCP servers and Man-in-the-Middle attacks. In this design, I define trust boundaries based on the principle of Administrative Control. By default, once the ip dhcp snooping command is enabled, all interfaces on a switch are considered untrusted. I then use the ip dhcp snooping vlan 10,20,30 command to specifically tell the switch to inspect and build a binding table for the traffic flowing within our Executive, Operations, and Lobby segments. Additionally, I used the “no ip dhcp snooping information option” command across the infrastructure to prevent DHCP lease failures caused by relay metadata in the lab environment.
  
On the access switches, I configured the uplink port (F0/1) as trusted. This port is under our direct control and forms the verified path leading back toward the authorized DHCP server on R1. By trusting only this specific uplink, we ensure that legitimate DHCPOFFER and DHCPACK messages can reach the clients, while all user-facing ports (F0/5-24) remain untrusted to block any rogue servers at the network edge. On the Layer 3 Switch, I marked the routed uplink toward R1 (G0/1) as trusted, as it is the direct entry point for authorized DHCP traffic. However, I have purposefully left the downlinks toward the Access Switches (F0/1-3) as untrusted. While these downlinks connect to managed switches that could be be configured as trusted as the traffic is coming from switches that are checking traffic, treating them as untrusted ensures that even if an access switch is compromised or a rogue server is somehow introduced into an intermediate trunk, the Layer 3 switch will still intercept and drop those unauthorized packets. This makes it so that the only truly "trusted" source in the entire building is the core uplink leading to R1. Now, if a rogue DHCP server attempts to send a fake DHCP Offer packet from an untrusted port, the switch immediately discards the traffic before it reaches any clients.
</p>
<p>
<img width="706" height="165" alt="image" src="https://github.com/user-attachments/assets/576b4665-d07a-4921-9204-9313d94683fa" />
</p>
<p>
<img width="694" height="135" alt="image" src="https://github.com/user-attachments/assets/d2aef128-29ee-483b-937c-df36e171b45e" />
</p>
<p>
<img width="693" height="136" alt="image" src="https://github.com/user-attachments/assets/42f43070-9112-4dce-8d9d-73298037226d" />
</p>
<p>
<img width="708" height="137" alt="image" src="https://github.com/user-attachments/assets/bc54e0b7-8b97-4a14-8863-5f4cc4f42f35" />
</p>
<br>

<p>
- Next, to mitigate ARP poisoning, I enabled Dynamic ARP Inspection (DAI) across both the L3SW and all access layer switches (SW1, SW2, and SW3). DAI operates at Layer 2 by inspecting ARP packets and validating their IP-to-MAC bindings against the DHCP Snooping Binding Table. When a device sends an ARP packet, each switch compares the sender’s IP and MAC address against this trusted database. If an attacker in the Lobby attempts to send a gratuitous ARP to spoof the gateway, the switch detects that the mapping does not match a valid DHCP snooping entry and discards the packet, preventing Man-in-the-Middle attacks and traffic redirection.
I configured the uplink interfaces on each access switch toward the L3SW as trusted, as they carry legitimate inter-switch traffic and validated DHCP Snooping information. However, all access-layer-facing ports connected to end-user devices in the Executives, Operations, and Lobby VLANs remain untrusted to ensure all ARP traffic originating from hosts is validated before being forwarded. DAI also supports the “ip arp inspection limit rate” command, which restricts the number of ARP packets a device can send per second to help mitigate ARP flooding attacks. However, since ARP rate limiting is not being tested as part of this project (as Packet Tracer doesn’t have an accurate way to test this), I don’t configure the command. DAI also supports the “ip arp inspection validate” command, which adds additional verification checks beyond the DHCP Snooping Binding Table. These include validating the consistency of the source MAC address, destination MAC address, and IP address within ARP packets. This ensures that even if an ARP packet appears structurally valid, it is still inspected for mismatches that could indicate spoofing or manipulation attempts before being forwarded through the network. Because this is beyond the scope of this lab, it won’t be configured either.
</p>
<p>
<img width="608" height="82" alt="image" src="https://github.com/user-attachments/assets/c4629c35-d79e-4018-ad64-672d14c36586" />
</p>
<p>
<img width="596" height="80" alt="image" src="https://github.com/user-attachments/assets/b7b61e8b-e80c-4dd8-8f56-ad7ca508b46a" />
</p>
<p>
<img width="593" height="80" alt="image" src="https://github.com/user-attachments/assets/3cc8fb39-334e-4e47-94a9-874eda6b5f5f" />
</p>
<p>
<img width="597" height="84" alt="image" src="https://github.com/user-attachments/assets/aec41a9a-3b9e-4aa3-a0e5-2e8770a249f7" />
</p>
<br>

<p>
- Onto the Access List (ACL) configurations, I create a Standard Numbered ACL that controls which source networks are allowed to reach the Executives VLAN. Standard ACLs only evaluate the source IP address, making them lightweight and efficient for destination-focused filtering. Because VLAN-to-VLAN filtering inside the local enterprise network is relatively straightforward and only requires filtering based on source subnet identity, Standard ACLs are the better design choice here. They are simpler, consume fewer router resources, and are easier to manage when only basic segmentation decisions are needed. To keep it even more simple and stripped down, I will be using numbered ACLs for these VLAN-to-VLAN ACLs, rather than named ACLs which require extra steps and router resources. The goal of these ACLs is to  reinforce the hierarchical trust model of the enterprise. Executives will maintain broad organizational access, while Operations remains more tightly restricted. By preventing Operations devices from communicating with the guest network, the attack surface for potential compromise is significantly reduced.
  
When creating ACL entries, Cisco requires either a permit or deny action to define whether traffic should be allowed or blocked. After specifying the action, the source network is defined followed by a wildcard mask. A wildcard mask is essentially the inverse of a subnet mask; instead of identifying the network portion that must match, it identifies which bits the router can ignore during the comparison process. For example, a wildcard mask of 0.0.0.255 tells the router to exactly match the first three octets while allowing any value in the final octet. This is what allows an ACL to represent an entire subnet rather than requiring individual host entries for every device. 
  
For the first VLAN-to-VLAN ACL, I configure policy controls for traffic entering the Lobby VLAN. Executive users are permitted communication with the Lobby because Executives require visibility and access across all business areas within the organization. However, Operations users are intentionally excluded from the Lobby network to reduce unnecessary lateral movement opportunities between medium-trust and low-trust zones. Additionally, to prevent the return paths of server architecture from being dropped by the filter structure, I include explicit permit declarations for the enterprise server blocks at the top of the sequence. These two new rules explicitly permit the 192.168.1.0/24 and 192.168.2.0/24 server networks to send traffic out of this interface. This ensures that when external servers respond to a client inside this VLAN, the return packet matches a permit rule and bypasses the implicit deny, resolving the silent drop issue in the simulation engine. Later on there will be other ACLs handling the permissions to the servers.
  
Because Standard ACLs can only evaluate source addresses, Cisco best practice is to place them as close to the destination as possible. This prevents unintentionally blocking traffic that may still need to reach other networks. So I applied the ACL outbound on the Lobby VLAN subinterface so packets are evaluated directly before entering the untrusted segment. An outbound ACL evaluates packets after the router has already made its routing decision and as traffic exits the router interface toward the destination network. This ensures unauthorized traffic is stopped immediately before reaching the protected VLAN.
</p>
<p>
<img width="774" height="166" alt="image" src="https://github.com/user-attachments/assets/d5f0f0a5-4607-4b8b-8156-08bbcf3cb492" />
</p>
<br>

<p>
- For the next ACL, the Operations department is permitted communication with the Executives VLAN because Operations users require controlled access to executive resources for organizational workflows. However, the Lobby VLAN is intentionally excluded because it represents an untrusted guest segment. This creates a layered trust model where Operations occupies a middle-trust tier between Executives and external guests.
  
Then, I implement tighter segmentation between the Lobby VLAN and the Executives VLAN. Because the Lobby is considered the lowest-trust network in the enterprise, unrestricted communication toward Executive systems would not follow with our model. However, business requirements still require guests to communicate with one designated Executive workstation for reception and coordination purposes. To accomplish this, I use the host keyword to permit access toward only a single Executive device at 100.1.1.11. Since ACLs contain an implicit deny at the end, all other Executive hosts are automatically blocked without requiring additional deny statements. This creates a secure "pinhole" policy where the Lobby VLAN gains access to only one carefully controlled endpoint rather than the entire Executive department. 
  
Like in the previous step, I also add extra ACEs for permit parameters covering both active server subnets at the top of the list. These two extra entries permit traffic sourcing from 192.168.1.0/24 and 192.168.2.0/24 to pass through the outbound interface cleanly. This ensures that server-to-client return traffic is exempted from the standard internal VLAN blocks, allowing legitimate web and domain replies to clear the interface security check without being caught by the implicit deny. 
  
It is also considered best practice to increment numbered ACLs by 10 rather than sequentially by 1. This is why the first ACL uses access-list 10 while this ACL uses access-list 20. By leaving gaps between ACL numbers, administrators gain flexibility for future expansion because additional ACLs can later be inserted logically between existing policies without disrupting the numbering structure. This improves long-term manageability in enterprise environments where ACL requirements frequently evolve over time. Standard ACLs use the range 1-99 and only filter traffic based on source IP addresses, making them ideal for simpler local segmentation policies like VLAN-to-VLAN filtering. Extended ACLs use the ranges 100-199 and 2000-2699 and provide much deeper inspection capabilities, including source addresses, destination addresses, protocols, and port numbers.
ACLs themselves are always processed from top to bottom in sequential order. As soon as traffic matches a statement, the router immediately performs the action and stops processing additional entries. At the end of every ACL is an invisible implicit deny statement that automatically blocks all traffic that does not explicitly match a permit rule. This is why only the single permitted Executive host can be reached while all other Executive devices are automatically denied without additional configuration.
  
The ACLs are again applied outbound as close to the destination as possible, this time on the g0/0.10 subinterface so unauthorized guest traffic is discarded immediately before entering the destination subnet. Additionally, ACL 30 is created for the sole purpose of giving server access to the G0/0.20 subinterface like we did for the other subinterfaces.
</p>
<p>
<img width="774" height="189" alt="image" src="https://github.com/user-attachments/assets/dc601a72-a209-4eb4-b3c3-e795f00886e0" />
</p>
<p>
<img width="771" height="133" alt="image" src="https://github.com/user-attachments/assets/cbaf7688-d0ed-429d-a708-c6493066571e" />
</p>
<br>

<p>
- Next is the server-access ACLs, where I will be using Extended Named ACLs. Unlike Standard ACLs, Extended ACLs can filter based on source address, destination address, protocol type, and transport-layer port numbers. This makes them ideal for application-aware filtering where specific services such as HTTPS or DNS must be selectively permitted.
  
While Standard ACLs work well for simpler local VLAN segmentation policies, server security requires more granular control. For external-facing services and sensitive infrastructure resources, we need the ability to inspect protocols and services directly. Extended ACLs provide this granularity by allowing filtering decisions based on TCP, UDP, and specific application ports. This gives the administrator precise control over exactly which services each VLAN may access.
And the reason to use Named ACL configuration is the increased flexibility it provides for long-term ACL management. With traditional numbered ACL configuration, normally you cannot directly edit or remove individual ACEs (Access Control Entries); instead, you must delete and recreate the entire ACL if changes are required. However, Named ACLs have their own configuration mode you enter, where you are able to add ACEs. This allows the direct modification or removal of individual entries without rebuilding the full ACL. Cisco also allows numbered ACLs to be edited using Named ACL configuration mode, which provides additional management flexibility even when working with numbered ACLs. This also ties into the earlier discussion about ACL sequencing.
  
Centralizing all server ACLs on R1 is also a stronger management and security design. Instead of distributing security policies across multiple routers, all server protections are enforced from a single controlled location. This simplifies troubleshooting, reduces configuration inconsistency, and creates a centralized enforcement point for enterprise security policy.
Onto the ACL configuration itself, 3 different extended ACLs must be created. For the high-trust Executives subinterface (G0/0.10), I create a profile named EXEC_INBOUND. The HTTPS server (192.168.1.100) is configured so that the Executives VLAN is allowed access to both HTTP (TCP port 80) and HTTPS (TCP port 443). To make this a single unified policy table, I combine these web rules directly with their required DNS rules (UDP port 53) within the same list. For the DNS entry, I explicitly define the service using the keyword “domain” instead of the protocol number, to showcase that Extended ACLs allow you to use either the number or a keyword for select protocols. At the end of the ACL, I implement a permit ip any any statement. This catch-all permits any traffic that has not matched a previous permit or deny entry, allowing normal routing and communication to continue for destinations not explicitly restricted by the policy. 
</p>
<p>
<img width="773" height="97" alt="image" src="https://github.com/user-attachments/assets/e52921a0-3f9e-49c5-8843-54388d855e06" />
</p>
<br>

<p>
- The other 2 extended ALCs, are for the medium-trust Operations segment and the untrusted Lobby guest zone. For the Operations subinterface (G0/0.20), I make the OP_INBOUND ACL. Operations users are intentionally denied access to the secure management web server, as in this scenario operations users are not required or expected to use HTTP/HTTPS. To achieve this safely, I explicitly permit their necessary DNS utility lookups to the central name resolution server on UDP port 53, and follow it immediately with explicit deny ip blocks targeting the servers. This design ensures that Operations can communicate with the DNS service, but cannot touch any other protocol or access the HTTPS web server. A final permit ip any any statement is placed at the end to allow their remaining daily business and general routing paths to traverse the gateway unchecked. Extended ACLs are particularly useful here because DNS filtering requires inspection of both the transport protocol and the application service itself. A Standard ACL would not be capable of differentiating DNS traffic from other traffic types because it only evaluates source addresses. Extended ACLs provide the necessary visibility into UDP services, allowing the router to selectively permit DNS traffic while denying everything else. 
  
For the Lobby guest subinterface (G0/0.30), I configure the LOBBY_INBOUND ACL to maintain strict public isolation boundaries. The Lobby VLAN is permitted access to the HTTPS web server on ports 80 and 443 so guests can open the public-facing corporate portal. However, the Lobby VLAN is intentionally denied DNS access because guest systems should not have visibility into internal enterprise resources or naming infrastructure. To enforce this, I place a hard deny ip rule targeting the DNS server directly beneath the allowed web lines, followed by a block to the rest of the HTTPS server fabric, before adding the final broad permit ip any any rule to let guests browse the open internet.
</p>
<p>
<img width="776" height="96" alt="image" src="https://github.com/user-attachments/assets/9bb8e7df-4190-46f6-9d3e-8eda811803c4" />
</p>
<p>
<img width="775" height="119" alt="image" src="https://github.com/user-attachments/assets/fe44fdd6-08bc-489b-af13-9b136c48e7fc" />
</p>
<br>

<p>
- In this final ACL deployment step, I apply the EXEC_INBOUND, OP_INBOUND, and LOBBY_INBOUND ACLs inbound on their respective client-facing subinterfaces. This ensures that the router inspects all traffic immediately as it enters the gateway from the local subnets, allowing us to drop unauthorized packets instantly before any routing or processing resources are wasted on the transit link.
</p>
<p>
<img width="678" height="163" alt="image" src="https://github.com/user-attachments/assets/2928b6c4-faa4-4db0-bfbc-65adf8c7c49d" />
</p>
<br>

<p>
- Now that all ACLs have been created and applied, I can confirm them by using the “show access-lists” command. Verification that all Access Control Lists have been successfully created and correctly applied to their respective interfaces, this is critical because an ACL that exists but is not properly applied provides no actual security enforcement.
  
First, I use the show access-lists command to display all configured ACLs on the router. This allows me to verify the ACL names, sequence of entries, permit and deny statements, and packet match counters. Cisco also supports the show ip access-lists command, which specifically filters and displays only IP-based ACLs. This is particularly useful in larger enterprise environments where multiple ACL types may exist and administrators need to isolate IP filtering policies for troubleshooting or auditing purposes.
  
Next, I use the “show ip interface” command to verify which ACLs are actively bound to each interface and in which direction they are operating. For example, interface g0/0.10 shows outbound access-list 20 and inbound access-list EXEC_INBOUND. This confirms that the router is correctly enforcing separate security policies for traffic entering and leaving the interface. You can also show the output of only a single interface (instead of all interfaces) by specifying the interface with the show ip interface command.
</p>
<p>
<img width="775" height="605" alt="image" src="https://github.com/user-attachments/assets/51fdc949-30b0-4e84-8e4e-60390322d9b1" />
</p>
<p>
<img width="773" height="309" alt="image" src="https://github.com/user-attachments/assets/c017893a-98f7-47d5-97f1-303101daa52a" />
</p>
<br>

<p>
- Next up, I extend the physical security hardening to the server-side of the infrastructure, specifically targeting SW4 (HTTPS Server connection) and SW5 (DNS Server connection). I configure the switch ports connected to these servers with Port Security to lock them down to the known MAC addresses of the server NICs. Because these servers are critical assets, we cannot allow rogue hardware to be plugged into these ports. By using sticky learning, the switch memorizes the server's identity. If anyone attempts to intercept the connection or replace a server with unauthorized equipment, the port will enter an err-disabled state immediately.
</p>
<p>
<img width="773" height="156" alt="image" src="https://github.com/user-attachments/assets/27e86d37-ce9d-4732-8aa7-37ca25c41ebe" />
</p>
<p>
<img width="769" height="156" alt="image" src="https://github.com/user-attachments/assets/35767f46-642c-42b1-bfff-ce2795deafaa" />
</p>
<br>

<p>
- As the last security measure, I configure the server-facing ports on SW4 and SW5 as Trusted for both DHCP Snooping and Dynamic ARP Inspection. Since these servers are managed static assets we have management control over, we permit these interfaces to send traffic without the same rigorous inspection applied to guest or user ports. I enable DHCP snooping and DAI on VLAN 1 as these switches are operating with default settings, meaning all active ports natively belong to the default Broadcast Domain of VLAN 1. However, I am keeping all other unassigned ports on these switches as Untrusted. This configuration ensures that if an attacker finds an open jack in the server side and plugs in a laptop, they will be blocked by DHCP Snooping and DAI by default, as their traffic won't be in the binding table and the port isn't marked as trusted.
</p>
<p>
<img width="693" height="191" alt="image" src="https://github.com/user-attachments/assets/3bdba3f5-557a-451a-9cfd-319c167f2899" />
</p>
<p>
<img width="691" height="191" alt="image" src="https://github.com/user-attachments/assets/c881a2ce-14c0-4d11-ac88-9ddd783fa7c0" />
</p>
<br>

<p>
- Now that all configuration is complete across the networks, I can begin testing all the security measures to make sure it all is working as it should. Before doing so, here is an updated view of the topology, with added IP address labels for each of the PCs, serving as a reference for the different tests to be better understood.
</p>
<p>
<img width="777" height="353" alt="image" src="https://github.com/user-attachments/assets/d51478b4-03da-4f33-b71e-c359c5c5b633" />
</p>
<br>

<p>
- To verify the standard ACL 10 configuration applied outbound on the Lobby VLAN interface (G0/0.30), I conduct cross-VLAN communication tests to confirm that traffic segmentation rules are active. First, I execute a ping from PC1 in the Executives VLAN (100.1.1.11) toward PC5 in the Lobby VLAN (100.3.3.11). This ping succeeds because the router evaluates the packet's source IP address against access-list 10, identifies an explicit match in the permit statement for the 100.1.1.0/24 subnet, and allows the traffic to leave the interface.
</p>
<p>
<img width="774" height="461" alt="image" src="https://github.com/user-attachments/assets/98959d89-1411-41e9-bd83-7fe8ee936393" />
</p>
<br>

<p>
- Next, I execute a ping from PC3 in the Operations VLAN (100.2.2.11) toward the same Lobby destination host. This ping fails, resulting in a destination host unreachable message. The failure occurs because the Operations source subnet is not explicitly permitted anywhere within access-list 10, causing the packet to hit the implicit deny statement at the end of the ACL and confirming that unauthorized lateral movement into the guest segment is blocked.
</p>
<p>
<img width="773" height="384" alt="image" src="https://github.com/user-attachments/assets/67f5227b-cf48-4c9e-8177-c00ea7024c85" />
</p>
<br>

<p>
- To validate the standard ACL 20 configuration protecting the Executives VLAN subinterface (G0/0.10), I do a ping from PC3 in the Operations VLAN (100.2.2.11) to PC2 in the Executives VLAN (100.1.1.12). This ping succeeds because the first ACE explicitly permits the entire 100.2.2.0/24 subnet to flow into the Executive environment.
</p>
<p>
<img width="775" height="421" alt="image" src="https://github.com/user-attachments/assets/a7252c6d-68ab-49bf-8b56-dc165bc59481" />
</p>
<br>

<p>
- I then test guest connectivity by running a ping to PC1’s 100.1.1.11 from the Lobby host PC5 with IP address 100.3.3.11, that has access to communicate with the Executive subnet. The ping is successful because the router finds a direct match on the second ACE (permit host 100.3.3.11), allowing this single trusted guest device through. In this example, this works because it could be that the first usable IP address of the pool (which PC is using) is always configured to be leased to a trusted device, perhaps PC5 is a personal computer that is being used by a trustworthy administrative lobby staff member.
</p>
<p>
<img width="773" height="403" alt="image" src="https://github.com/user-attachments/assets/5434f90d-2b75-4dd0-addb-226ba3c4a658" />
</p>
<br>

<p>
- Finally, I run a ping from PC6 in the Lobby VLAN configured with 100.3.3.12 to Executive VLAN’s PC2 with 100.1.1.12. This final attempt fails and times out because standard ACL processing stops after failing to match the explicit Operations subnet or the unique 100.3.3.11 host statement, forcing the unauthorized guest traffic to be discarded by the implicit deny rule.
</p>
<p>
<img width="772" height="379" alt="image" src="https://github.com/user-attachments/assets/6c2bf2d0-e9e0-475a-b617-e9c59a659288" />
</p>
<br>

<p>
- To test the core security controls of the EXEC_INBOUND and LOBBY_INBOUND Extended ACLs applied on R1, I initiate web traffic verification tests from different corporate layers targeting the HTTPS Server (192.168.1.100). First, I attempt to access the web service from PC1 in the Executives VLAN (100.1.1.11) using the web browser. The application loads successfully over HTTPS (port 443). I repeat this test from PC5 in the Lobby VLAN (100.3.3.11), but this time with HTTP (port 80), which succeeds.
  
This validates that the EXEC_INBOUND and LOBBY_INBOUND lists correctly grant web access to authorized tiers because of the explicit Layer 4 tcp port 80 and 443 rules, Executives require access to web management platforms, and guests must be allowed to open the central public-facing portal over safe internet protocols.
</p>
<p>
<img width="777" height="255" alt="image" src="https://github.com/user-attachments/assets/772d4f20-408c-49bb-a920-c34ea9710a4a" />
</p>
<p>
<img width="777" height="222" alt="image" src="https://github.com/user-attachments/assets/eca2bf95-9fd3-4d68-a3ce-79560bbeea27" />
</p>
<br>

<p>
- To verify the enforcement boundaries of the consolidated OP_INBOUND filter, I attempt a web connection from PC3 inside the restricted Operations VLAN (100.2.2.11) toward the HTTPS Server (192.168.1.100). The request results in a timeout error in the browser interface. Because the Extended Named ACL intercepts inbound traffic on subinterface g0/0.20 and checks for matching TCP parameters, the session immediately drops at the router inbound boundary because of the explicit server deny entry, proving that unauthorized application-layer access strings are actively dropped. The reason we want this blocked is because Operations users focus strictly on back-end utility functions, they have no baseline business requirement to browse the front-facing web portal, allowing us to proactively deny it.
</p>
<p>
<img width="778" height="188" alt="image" src="https://github.com/user-attachments/assets/7c8de151-8715-4a61-9139-a2ee65448f08" />
</p>
<br>

<p>
- To validate the DNS permissions of the consolidated EXEC_INBOUND and OP_INBOUND , I perform verification tests across multiple subnets. From the command prompt of PC1 in the Executives VLAN (100.1.1.11), I run a name resolution ping command targeting R1 by pinging the router’s hostname “R1”, instead of using the regular IP address.The lookup succeeds, translating the router’s name to its mapped destination. I repeat this validation step successfully from PC3 in the Operations VLAN (100.2.2.11). The successful name translation confirms that the network safely resolves critical resources across the internal routing fabric. The reason we want these lookups to pass cleanly is because internal naming resolution via the central domain controller is a vital operational utility required by both internal workers and system administrators to trace network hardware, reach core routers, and handle standard daily connectivity workflows without having to always type raw IP addresses. The 3rd screenshot shows the same ping, this time from PC5 in the Lobby VLAN. The translation utility immediately stalls and prints a resolution error, proving that guest nodes are blocked from polling the environment. The reason we explicitly constructed this block in Step 14 is to protect our internal infrastructure footprint; allowing public guest machines to resolve hostname records for critical assets like R1 and R2 would grant potential attackers visibility into corporate naming and internal routing layouts.
</p>
<p>
<img width="773" height="378" alt="image" src="https://github.com/user-attachments/assets/beb6d740-815c-4d7f-a1a6-6b8fb9079907" />
</p>
<p>
<img width="774" height="387" alt="image" src="https://github.com/user-attachments/assets/864f4d06-5282-47a3-a297-2cfe8cb59740" />
</p>
<br>

<p>
- To confirm that public visitor devices cannot gain visibility into internal enterprise naming architecture, I test the negative boundary of the LOBBY_INBOUND filter from PC5 in the Lobby VLAN (100.3.3.11). When executing the identical name resolution command toward R1, it fails and logs a name resolution error because of our explicit entry blocking the guest network from reaching the DNS server, proving that the security hole has been successfully closed. The reason we explicitly constructed this block is to protect our internal infrastructure footprint; allowing public guest machines to resolve hostname records for critical assets like R1 and R2 would grant potential intruders visibility into corporate naming schemas and internal routing layouts, which completely violates our commitment to an aggressive defense-in-depth security model. 
To finish the ACL tests, I run the “show access-lists” command. Now that several tests across the subnets have been ran, we can see the ACLs now display matches, showing that they are successfully filtering traffic that is related to each ACL. This is very useful for troubleshooting issues, to see which ACLs are operational, and to monitor traffic.
</p>
<p>
<img width="773" height="185" alt="image" src="https://github.com/user-attachments/assets/09cf90aa-b59e-466a-bbe6-750a7f2f74db" />
</p>
<p>
<img width="768" height="536" alt="image" src="https://github.com/user-attachments/assets/d7087a39-db72-4d15-bca4-4a68215f8f52" />
</p>
<br>

<p>
- For the first Layer 2 test, I will test the performance of the Port Security configuration on our access layer by simulating a MAC spoofing attack. This defense mechanism is vital for stopping unauthorized edge entry, which is a major real-world risk where an intruder disconnects a legitimate workstation and plugs in an unauthorized laptop to bypass network boundaries. Port security enforces a hard MAC-to-port pinning strategy that ensures only verified corporate assets can inject frames into the switching fabric.
  
To carry out this test, I click on the connection linking Executive PC1 to SW1, and delete the cable. I then add Laptop 1, and establish a new copper straight-through cable connection running directly from the laptop's interface F0/1 into interface F0/2 on SW1. Because an unconfigured interface cannot transmit valid frames, I open the Laptop's settings, navigate to the IP configuration tab, and toggle the setting to DHCP. This action forces the laptop to start the DORA process and broadcast a DHCP Discover frame. The moment this frame hits the switch interface, SW1 identifies the unauthorized source MAC address, triggers a port-security violation, and shifts the physical port link light to solid red. I then open the SW1 CLI console and verify that a port-security violation error Syslog message has been generated, proving that the switch successfully locked down the port and neutralized the unmapped device. By forcing the port into an err-disabled state upon receiving an unmapped frame, we actively neutralize the attack string before the rogue host can execute reconnaissance scripts, scan internal segments, or sniff active broadcast streams.
  
If I were to want to restore the err-disabled interface to an operational state, there are two recovery methods I could use. The first is manual recovery, where I would enter the affected interface and issue the “shutdown” command followed by “no shutdown”, resetting the port and clearing the err-disabled condition. The second is automatic recovery through Cisco's ErrDisable Recovery feature, which can be configured to automatically re-enable interfaces disabled by specific violation types after a defined timer expires. ErrDisable is not limited to Port Security violations; it can also be triggered by several other security and protection mechanisms such as BPDU Guard, Power Policing, Dynamic ARP Inspection (DAI), and other interface protection features. It is important to first identify and correct the underlying cause of the violation before restoring it; otherwise, the interface will simply enter the err-disabled state again as soon as the switch detects the same problem.
</p>
<p>
<img width="787" height="526" alt="image" src="https://github.com/user-attachments/assets/f6e4534c-6582-460e-906c-ad4a3484a458" />
</p>
<p>
<img width="778" height="215" alt="image" src="https://github.com/user-attachments/assets/e56024d8-e5c6-4c3b-967c-e53c3f914905" />
</p>
<p>
<img width="789" height="548" alt="image" src="https://github.com/user-attachments/assets/6d973ab5-25bf-44da-b49a-34ae03cf8175" />
</p>
<p>
<img width="778" height="128" alt="image" src="https://github.com/user-attachments/assets/04f8bf89-8e85-4ab2-82a4-a6c10781ebf4" />
</p>
<p>

</p>
<br>

<p>
- In the next test, I will check edge trust boundaries enforced by DHCP Snooping by simulating a rogue DHCP server injection attack on an access switch port. This vulnerability represents a critical infrastructure threat where an attacker introduces a rogue dynamic provisioning system to lease out corrupted IP configuration metadata. If left unchecked, clients accepting these rogue parameters could be silently redirected to a malicious gateway, facilitating massive Man-in-the-Middle hijacking and credentials harvesting schemes. By designating user-facing interfaces as strictly untrusted, the access switch acts as an intelligent signaling filter that immediately parses and drops unauthenticated inbound server frames.
  
To execute this validation, I add a new generic server to simulate an attacker's rogue DHCP platform, connect it to SW2’s F0/15 port, which is configured as an untrusted edge port. I assign the rogue server an IP address of 100.2.2.200/24 within the Operations VLAN, enable the DHCP service, and create a malicious DHCP scope targeting the Operations subnet. The scope is intentionally configured to advertise the attacker's own address (100.2.2.200) as both the Default Gateway and DNS Server instead of the legitimate gateway (100.2.2.254) and correct DNS server. This simulates a real-world rogue DHCP attack where an attacker attempts to convince clients that all outbound traffic should be sent through the attacker's system. If a client accepts these settings, every packet destined outside the local subnet would first be forwarded to the attacker's machine, allowing the attacker to inspect, log, modify, or redirect traffic before passing it along to its intended destination.
  
To verify the blocking functionality, I move over to Operations PC3, open its command prompt, and force a dynamic reconfiguration sequence by entering “ipconfig /release” followed immediately by “ipconfig /renew”. I monitor the packet exchange and confirm that PC3 never accepts the rogue DHCP offers and instead maintains its legitimate router-assigned scope from R1, proving that SW2 successfully filtered the unauthorized DHCP server messages at the interface boundary. The terminal results confirm that the unauthorized lease distribution system is completely isolated at the edge, guaranteeing that internal nodes retain their legitimate, administrator-controlled configurations without interruption.

Because we previously configured DHCP Snooping on SW2 and designated all user-facing access ports as untrusted, the switch automatically inspects DHCP traffic and drops any DHCPOFFER or DHCPACK messages arriving from those interfaces. Since only the uplink leading toward the authorized DHCP server on R1 was marked as trusted, the rogue server's responses never reach PC3, while legitimate DHCP traffic continues to pass normally. By dropping DHCP server messages arriving from untrusted interfaces, DHCP Snooping prevents clients from ever receiving these rogue configurations and preserves the integrity of the trusted infrastructure.
</p>
<p>
<img width="782" height="503" alt="image" src="https://github.com/user-attachments/assets/5d3956fe-19d7-4e1b-9785-914ba828bbc8" />
</p>
<p>
<img width="780" height="275" alt="image" src="https://github.com/user-attachments/assets/2bb0262a-0e5a-4ae0-8b05-5b8e99a766c4" />
</p>
<p>
<img width="780" height="326" alt="image" src="https://github.com/user-attachments/assets/a739b274-76c3-43f6-b38a-a86d7009047b" />
</p>
<p>
<img width="774" height="316" alt="image" src="https://github.com/user-attachments/assets/6db342a6-0eea-49e3-b83f-ce133aafc3a2" />
</p>
<br>

<p>
- In this test, I will verify the security performance of Dynamic ARP Inspection (DAI) against local network address hijacking and gateway spoofing attacks on the SW3 Access Layer Switch. An attacker connected to an untrusted port may attempt an ARP poisoning attack by sending fraudulent ARP messages that associate the IP address of the default gateway with the attacker's own MAC address. If successful, hosts update their ARP tables with the incorrect mapping and begin forwarding traffic through the attacker’s device, enabling a Man-in-the-Middle attack that allows interception, monitoring, and manipulation of local traffic.
  
DAI prevents this by inspecting all ARP packets arriving on untrusted interfaces and validating their IP-to-MAC mappings against the DHCP Snooping Binding Table created earlier in the project. Because this table only contains verified DHCP-learned bindings, any ARP packet claiming ownership of an IP address that does not match the recorded MAC address is immediately dropped. This creates a dependency where DHCP Snooping establishes trust and DAI enforces it at Layer 2.

To demonstrate this protection mechanism, I add Laptop 2 into the lab, connect it to F0/15, an untrusted access interface on SW3, the access layer switch for the Lobby VLAN. I then manually assign the laptop with 100.3.3.254 for its IP address (as well as the default gateway and DNS server), which represents the default gateway for the Lobby VLAN. This intentionally creates an IP-to-MAC mismatch because the laptop’s MAC address does not match the gateway MAC address stored in the DHCP Snooping Binding Table. I then generate traffic from the laptop by pinging PC5 to trigger ARP activity on the network. As the spoofed ARP packets reach the switch, DAI compares the sender information against the trusted DHCP Snooping Binding Table and immediately drops any packet that fails validation. The failed ping indicates that communication to the target network was not successfully established, suggesting that ARP resolution and IP-to-MAC mapping were not completed due to the configured security controls, preventing potential gateway spoofing from affecting legitimate hosts.
  
If DAI were not enabled, hosts in the Lobby VLAN could accept the attacker’s fraudulent ARP advertisements and associate the gateway address 100.3.3.254 with the attacker’s MAC address. Traffic intended for the legitimate gateway would then be redirected through the attacker’s device, allowing full interception, modification, or redirection of communications before forwarding them to the real gateway. By enforcing IP-to-MAC validation against the DHCP Snooping Binding Table and dropping mismatches, DAI prevents ARP cache poisoning and preserves the integrity of Layer 2 communications.
</p>
<p>
<img width="787" height="513" alt="image" src="https://github.com/user-attachments/assets/657609b3-9f58-40be-ae89-74f0f0c4b0b6" />
</p>
<p>
<img width="776" height="251" alt="image" src="https://github.com/user-attachments/assets/7a8b4d07-e7b6-4607-85dd-a148149b8c9b" />
</p>
<p>
<img width="776" height="387" alt="image" src="https://github.com/user-attachments/assets/e3ae9c93-9800-4c5f-89b6-22e1b309e2ea" />
</p>
<br>

<p>
- As a final verification, I perform a quick operational check of the physical port security protections applied to the server-facing switches SW4 and SW5. In enterprise environments, periodic validation of security controls is often sufficient to ensure that critical protections remain active and correctly configured, especially on stable infrastructure such as server access layers. This focuses on confirming that port security is functioning as expected on the interfaces connected to the servers.
  
To verify this check, I access the CLI on both SW4 and SW5 and run a status review using the “show port-security interface f0/1” command. This allows me to confirm that port security is enabled, the interface is in a secure operational state, and that the configured parameters (such as the violation mode and learned MAC address binding) remain correctly applied. Show commands are some of the most useful tools to use when troubleshooting and making sure security measures are operational. Since no violations are present, no corrective actions are required, and the existing configuration is confirmed to be stable and operational.
</p>
<p>
<img width="712" height="361" alt="image" src="https://github.com/user-attachments/assets/12ff27e2-b0b5-42e9-b4cb-bd239fb27d11" />
</p>
<p>
<img width="716" height="361" alt="image" src="https://github.com/user-attachments/assets/f2a984fd-6d63-4df6-886b-8a27cc6105e7" />
</p>
