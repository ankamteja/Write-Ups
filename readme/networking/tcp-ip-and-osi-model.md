# TCP/IP and OSI model

**Overview of Networking Models: TCP/IP and OSI**

In the early days of computing, devices from different manufacturers were unable to communicate because they utilized proprietary, incompatible networking designs. The development of standardized networking models solved this by creating a common language for devices to connect and share data.

**The TCP/IP Model**

* The **TCP/IP model** (or stack) is the industry standard used for modern networking.
* It acts as a collection of rules and guidelines that govern how systems are designed to communicate.
* To simplify the complexity of network functions, the model is divided into layers, with each layer handling specific protocols (e.g., Physical, Data Link, Network, Transport, and Application).

**The OSI Model**

* The **OSI model** (_Open Systems Interconnect_) consists of seven layers. While it was once intended to be the primary standard, it was ultimately surpassed by TCP/IP in practical adoption.
* Despite this, the OSI model remains the universal reference point used by engineers for daily communication and troubleshooting.
* Engineers frequently refer to the specific layers (e.g., "Layer 7" for Application, "Layer 3" for Network) when discussing or diagnosing networking issues.

**Key Differences and Terminology**

* While the models share similar concepts for the lower levels (Physical, Data Link, Network, and Transport), the OSI model separates the Application layer into three distinct layers: Application, Presentation, and Session.
* In the real world, network professionals typically map these extra OSI layers into the broader "Application" layer of the TCP/IP stack but continue to use OSI terminology for technical discussions.

**Memorization Tips**

* To remember the seven layers of the OSI model, use mnemonic devices:
  * From top to bottom: "All People Seem To Need Data Processing."
  * From bottom to top: "Please Do Not Throw Sausage Pizza Away."
  *

While the real world primarily utilizes the **TCP/IP model**, the **OSI model** is often used as a reference framework to visualize the process of data moving between devices.



**The Layers of Data Transmission**

* **Application Layer (Layer 7):** This is where user-facing applications (like web browsers) initiate communication using protocols such as **HTTPS**. The computer takes the user's request and prepares it to travel down through the protocol stack.
* **Transport Layer (Layer 4):** This layer dictates how data is moved. It primarily uses **TCP** (reliable, connection-oriented) or **UDP** (fast, less reliable). For web traffic, **Port 443** is typically used for secure **HTTPS** connections. A data unit at this layer is known as a **segment**.
* **Network Layer (Layer 3):** This layer handles **IP addresses** and routing. Routers use this information to determine the path the data needs to take to reach its destination. A data unit at this layer is called a **packet**.
* **Data Link Layer (Layer 2):** This layer deals with **MAC addresses** and physical hardware like switches. It ensures the data gets to the next physical stop on the network path. A data unit at this layer is called a **frame**.

**Key Concepts**

* **Encapsulation:** As data moves from the Application layer down to the Physical layer, each layer adds its own header information (like an envelope inside an envelope) to ensure the data is correctly handled and routed.
* **De-encapsulation:** When a device receives a frame, it performs the reverse process. It strips away the headers layer by layer to interpret the information contained in the packet, segment, and eventually the application data.
* **The Physical Path:** Data travels as electrical or optical signals across hardware components (cables, switches, and routers) until it reaches the final destination server, where the process is reversed so the original application request can be read.

\
&#x20;     The upper layers of the OSI model—Application, Presentation, and Session—and the Transport layer, detailing how they function when a user accesses a website like YouTube.

**The Upper Layers (Application, Presentation, Session)**

* **Application Layer:** Serves as the portal or interface for network-aware programs, such as web browsers or video games. It initiates communication requests.
* **Presentation Layer:** Prepares data for the application. Key functions include data formatting (e.g., ensuring files like HTML, JPG, or PDF are readable) and encryption (e.g., SSL) to secure data.
* **Session Layer:** Manages the communication session between the application and a remote server. It handles authentication and keeps the conversation active until the task is complete.

**The Transport Layer** This layer determines how data is transmitted between hosts, choosing between two primary protocols based on the application's needs:

* **TCP (Transmission Control Protocol):** Focuses on reliability. It uses a "three-way handshake" (SYN, SYN-ACK, ACK) to establish a connection and requires verification (acknowledgments) for every piece of data sent. If data is lost, it is retransmitted.
* **UDP (User Datagram Protocol):** Focuses on speed. It transmits data without checking for receipt or requiring acknowledgments. This is ideal for real-time applications like video streaming or online gaming where retransmitting missed data would be detrimental.

**Ports: Managing Multiple Services** Ports allow a single server to host multiple services simultaneously (e.g., web hosting, FTP, SSH).

* **Well-known Ports:** Reserved ports (0-1023) used for standard services, such as port 443 for HTTPS.
* **Ephemeral Ports:** Temporary ports chosen by the client machine to receive return data, allowing the user's computer to keep track of multiple active sessions (like multiple browser tabs) at once.

**Summary of the Process** When a user accesses a website, the application layer gathers the request, the presentation layer formats and secures it, and the session layer manages the connection. The transport layer then assigns ports and chooses the appropriate protocol (TCP for the webpage, UDP for video streaming) to ensure the data is delivered to the correct destination and service.

<br>

