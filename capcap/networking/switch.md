# Switch

**Key Concepts:**

* **Switches vs. Hubs:**
  * A **hub** is described as "dumb" because it repeats incoming electrical signals out to every connected port, leading to security risks and network traffic inefficiency.
  * A **switch** is "smart"; it learns the unique identifiers of connected devices and directs traffic only to the specific destination port, improving performance and security.
* **The Role of Addresses:**
  * **MAC Address (Layer 2):** Every device has a unique, "burned-in" MAC address. Switches use these to identify devices on the network.
  * **IP Address (Layer 3):** Used for end-to-end communication, but switches generally operate at Layer 2 and do not rely on IP addresses for forwarding decisions.
* **How Switches Work:**
  * Switches maintain a **CAM table** (_Content Addressable Memory_), which maps specific MAC addresses to physical switch ports. This allows the switch to know exactly where a device is located.
* **Terminology:**
  * When data moves across the network via a switch (Layer 2), it is technically referred to as a **frame**.
  * When data is processed at Layer 3 (involving IP addresses), it is referred to as a **packet**.
* **Wireless Access Points:**
  * While they act as an extension of the network, wireless access points often behave more like hubs by broadcasting traffic, which is why hardwired Ethernet connections are generally preferred for performance.

**Practical Skills:**

* **Cisco Packet Tracer:** A simulation tool used to visualize network traffic at the packet/frame level.
* **First Cisco CLI Command:**
  * To view the switch's learned MAC address-to-port mappings, use the command: `show mac-address-table`.
