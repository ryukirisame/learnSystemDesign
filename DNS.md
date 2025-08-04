1. What is DNS?
2. Namespace, Tree Hierarchy, TLD, SLD, Subdomain, Zone
3. Types of DNS servers, Example of each type of DNS server, who manages each DNS servers
4. Types of DNS records
5. DNS resolution process
6. Delegation
7. Caching, TTL
8. GeoDNS
9. Types of DNS queries: recursive, iterative and non-recursive
10. DNS Security Issues
11. DNS query using dig command, ipconfig /displaydns
12. How DNS works/integrates with CDN, load balancing, fail over, geo-load balancing
13. Difference between www and mail. is www default?
14. Anycast IPs


# What is DNS?
Every single computer (or server) on the internet is identified by their IP address (e.g., `142.250.182.110` for Google). When one computer wants to communicate with another, it uses the destination’s IP address to connect to it. 

However, remembering numeric IP addresses is difficult for humans. We prefer to use names like `www.google.com` instead. This is where the Domain Name System (DNS) comes in. DNS translates human‑friendly domain names into machine‑readable IP addresses. And because of this, its often called _the phonebook of the internet_.

# Domain Name Space
It's a complete set of possible domain names. A name space that maps each IP address to a unique name can be organized in two ways: flat or hierarchical.

## Flat Name Space
In a flat name space, each name is just a simple label that directly maps to an address. There’s no hierarchy or structure — it’s just a big list of unique names.

This works fine for small systems, but it quickly becomes a problem on a global scale. Without structure, the only way to prevent duplicates is to have one central authority keep track of all names.

That means:
- Scalability issues – Every new name must be checked against the global list.
- No delegation – You can’t split the responsibility among different organizations.
- Single point of failure – If the central authority goes down, the entire naming system is at risk.

This is why the Internet uses a hierarchical naming system instead.

## Hierarchical Name Space
In a hierarchical name space, each name is made of several parts. The parts are organized in an inverted-tree structure with the root at the top. 

The DNS hierarchy can have a maximum of 128 levels:
- Level 0 → Root
- Level 127 → Deepest possible node

<img width="603" height="228" alt="image" src="https://github.com/user-attachments/assets/15ffd2b7-87fc-4050-82f3-2577ac8b0f20" />


#### Labels
Each node in the tree has a label, which is a string containing up to 63 characters.

- The root label is an empty string (null label).
- All child nodes of the same parent must have **unique labels**. This uniqueness requirement ensures that every domain name within the DNS hierarchy is unique.

#### Domain Names
Each node in the DNS hierarchy has a domain name.
A full domain name is formed by combining the labels from a given node up to the root, separated by dots (`.`).
- The last label is always the root label (null string), which means a fully qualified domain name ends with a dot (`.`).
- This trailing dot represents the root and is typically omitted in everyday use, but it exists conceptually.

Example:
`www.example.com.`
- `www` → label of a host node
- `example` → label of the second‑level domain node
- `com` → label of the top‑level domain node
- `.` → root label (null string)


<img width="545" height="349" alt="image" src="https://github.com/user-attachments/assets/1ab54393-5486-434a-9bd2-60c1e580d2e1" />

#### Domains
A domain is simply a subtree of the domain name space. 
- The name of the domain is the name of the node at the top of that subtree.
- A domain can itself be divided into subdomains, which are represented by lower‑level branches of the tree.

<img width="588" height="269" alt="image" src="https://github.com/user-attachments/assets/b83cf62e-ec0c-4f6d-b5c5-10c0e80db33c" />

#### Fully Qualified Domain Name (FQDN) vs Partially Qualified Domain Name (PQDN)

**Fully Qualified Domain Name**: If a domain name is terminated by a null string, it is called a fully qualified domain name (FQDN). The name must end with a null label, but because null means nothing, the label ends with a dot. Example: `mail.example.com.`.

**Partially Qualified Domain Name**: If a domain name is not terminated by a null string, it is called a partially qualified domain name (PQDN). A PQDN starts from a node, but it does not reach the root. It is used when the name to be resolved belongs to the same site as the client. Here the resolver can supply the missing part, called the suffix, to create an FQDN. Example: `mail.example` (resolver may append `.com`).

### Parts of a Hierarchical Domain Name

In the hierarchical name space, a **fully qualified domain name (FQDN)** is made up of different levels, read from right to left:

1. **Top‑Level Domain (TLD)** – The highest level in the hierarchy, located just below the root (`.`). Examples include:
    * **Generic TLDs (gTLDs)**: `.com`, `.org`, `.net`
    * **Country‑code TLDs (ccTLDs)**: Uses two-character country abbreviations. Eg: `.in` (India), `.uk` (United Kingdom), `.jp` (Japan).
       * Some ccTLDs use second‑level labels for further categorization.
          * Example: The United States `.us` namespace can have state abbreviations as second‑level domains, such as `.ca.us` for California.
          * `uci.ca.us.` translates to:
             * `uci` → University of California, Irvine
             * `ca` → State of California
             * `us` → United States country code TLD
             * `.` → Root
     
    <img width="467" height="236" alt="image" src="https://github.com/user-attachments/assets/0a42e568-55f6-4d86-8b17-592e939950fa" />

      
2. **Second‑Level Domain (SLD)** – The part immediately to the left of the TLD. It is typically chosen by the domain owner and represents the main identity of the domain.
    * Example: In `example.com`, `example` is the SLD.
3. **Subdomain** – Any label that appears to the left of the SLD. Subdomains are used to organize different sections or services of a website.
    * Example: In `mail.example.com`, `mail` is a subdomain of `example.com`.
4. **Host Name** – The specific name of a device or service within a domain. In many cases, the subdomain also serves as the host name.
    * Example: `www` in `www.example.com` is often used as the host name for the web server.

 **Example Breakdown**

 ```
 www.mail.example.com.
 └── Root (.)
    └── Top‑Level Domain: com
         └── Second‑Level Domain: example
             └── Subdomain: mail
                 └── Host Name: www
 ```
Every level in this structure is separated by a dot (`.`), and the entire sequence — including the final dot for the root — forms the **fully qualified domain name (FQDN)**.

### Why we need hierarchical name space?
- Scalability – No single place needs to store all names.
- Delegation – Each part of the tree can be managed by different organizations.
- Uniqueness – No two domains at the same level can have the same name.


# Types of DNS Servers

1. Recursive Resolver
2. Root Name Server
3. TLD (Top-Level Domain) Name Server
4. Authoritative Name Server

<img width="1080"  alt="image" src="https://github.com/user-attachments/assets/8cb68834-f595-49f4-88c0-034543b941c1" />

## Recursive Resolvers (DNS Resolvers)

A recursive resolver — also called a DNS Resolver — is the DNS server that receives queries directly from client devices, such as your computer or smartphone. Its main role is to find the answer to the client’s query by contacting other DNS servers if needed.

If the resolver already has the answer stored in its cache (from a recent lookup), it can return it immediately. Otherwise, it will query other DNS servers — such as the Root Name Server, TLD Name Server, and Authoritative Name Server — to get the result.

Most users rely on the recursive resolver provided by their Internet Service Provider (ISP), but there are also public alternatives, such as:
- Cloudflare (1.1.1.1)
- Google Public DNS (8.8.8.8)
- OpenDNS (208.67.222.222)


## Root Name Server
- Root Name Server doesn't know the IP address of the domain. They only know which TLD name servers (e.g., .com, .org, .net) are responsible for that domain.
- A root server accepts a recursive resolver’s query which includes a domain name, and the root nameserver responds by directing the recursive resolver to a TLD nameserver, based on the extension of that domain (.com, .net, .org, etc.).
- Example: Recursive resolver asks: "What is the IP for www.example.com?"
  - Root server replies: “I don’t know the IP, but `.com` TLD servers can help you. Here are their IPs for `.com` TLD servers.”
- There are 13 **sets** of name servers known to every recursive resolvers. Please note that the 13 name servers are logical. For each one of them, they have multiple servers distributed around the globe.
- The root nameservers are overseen by a nonprofit called the Internet Corporation for Assigned Names and Numbers (ICANN).
- All the root name servers use anycast technology to provide redundancy and performance optimization. 


## TLD Name Servers
- TLD name servers stores information about all domains under a specific TLD (.com, .org, .net, .in, etc.).
- They don’t know your website’s IP either. They only know which authoritative name servers are responsible for that domain. So basically a `.com` TLD server will store authoritative name servers IP of all the domains that ends with `.com`.
- They store: NS records (name server records) telling where to find authoritative servers for domains under that TLD.
- Query to TLD server: “What is the IP of www.example.com?”
   - TLD Server Response: “I don’t know the IP. But I know which authoritative name servers handle `example.com`. Here’s their list.”
- When you query a `com` TLD server for `www.example.com`, it matches that request against its database, finds the entry for `example.com`, and returns the NS records.
- Management of TLD nameservers is handled by the Internet Assigned Numbers Authority (IANA), which is a branch of ICANN.


## Authoritative Name Servers
- An Authoritative Name Server is the DNS server that holds the actual DNS records (IP and other DNS records) for a domain.
- It is the final source of truth in the DNS lookup process.
- When a recursive resolver queries an authoritative server for a domain, the authoritative server returns the answer directly — typically the A record (IPv4), AAAA record (IPv6), or CNAME record for the requested host.
- Managed by the domain owner or their DNS hosting provider(e.g., Cloudflare, AWS Route 53, GoDaddy).
- Query to authoritative name server: “What is the IP for `www.example.com`?”
   - Authoritative server reply: “The IP for www.example.com is `93.184.216.34`.”

### Subdomain & Delegation
A subdomain is simply a label before your main domain (Second-Level Domain).
It is part of your domain’s DNS hierarchy.

- Examples:
   - `www.example.com` → subdomain = `www`
   - `mail.example.com` → subdomain = `mail`
   - `blog.shop.example.com` → subdomains = `blog` (of `shop`), and `shop` (of `example.com`)


By default, the authoritative name server will manage all the subdomains. However, we can configure this to delegate some specific subdomains to another authoritative server. So, Instead of your main authoritative servers answering, they redirect the resolver to different authoritative servers for that subdomain. The resolver then queries the delegated authoritative server for the final answer.

#### Default Case (No Delegation)
If `example.com` is managed by: `ns1.mydnsprovider.com`, then:
- `www.example.com` will be resolved by `ns1.mydnsprovider.com`
- `mail.example.com` will be resolved by `ns1.mydnsprovider.com`
- `anything.example.com` will be resolved by `ns1.mydnsprovider.com`

#### Delegation Case
How it works:
1. The `example.com` authoritative server stores NS records for `blog.example.com` pointing to the delegated authoritative server’s IPs.
2. When a recursive resolver asks the `example.com` server about `blog.example.com`, it responds with the NS records for that subdomain instead of an IP address.
3. The resolver then queries the delegated authoritative server for the final answer.

As for other subdomains like `www.example.com`, the main authoritative server will still serve the IP address.

#### Why would you delegate a subdomain?
- Different service provider manages it
   - Example: Your website is on AWS, but your email service is hosted by Microsoft 365 → you might delegate `mail.example.com` to Microsoft’s DNS.
- Different team manages a part of your domain
   - Example: Your dev team controls `dev.example.com` independently.
- Third‑party services
   - Example: A blog platform manages `blog.example.com`.

## Summary
So in one sentence:
- Root server → “Here’s who manages `.com` domains.”
- TLD server → “Here’s who manages `example.com`.”
- Authoritative server → “Here’s the IP for `www.example.com`.”


# Common DNS Record Types

| Record Type | Purpose | Example |
|-------------|---------|---------|
| A Record | Maps domain names to IPv4 addresses | `example.com` → `192.0.2.1` |
| AAAA Record	| Maps domain names to IPv6 addresses	| `example.com` → `2606:2800:220:1:248:1893:25c8:1946` |
| CNAME Record	| Creates aliases pointing to other domain names | `www.example.com` → `example.com` | 
| MX Record	| Specifies mail servers for email routing | `example.com` → `mail.example.com` |
| NS Record	| Identifies authoritative name servers for a domain	| `example.com` → `ns1.example.com` |
| TXT Record	| Stores text information for various purposes	| Used for SPF, DKIM, domain verification |
| PTR Record	| Performs reverse DNS lookups (IP to domain)	| `192.0.2.1` → `example.com` |




# DNS Resolution Process
<img width="1360" height="680" alt="image" src="https://github.com/user-attachments/assets/96a8c2b2-0ea5-4345-87a9-b3f5c652a0bb" />

When you type www.example.com into your browser, your computer needs to translate it into an IP address (e.g., 93.184.216.34). This happens in several steps:

## 1. User Makes a Request
You enter `www.example.com` into your browser’s address bar and press Enter.

## 2. Browser & OS Cache Check

* Browser checks its DNS cache. If found, return the IP.
* If not found, OS resolver cache is checked.
* If still not found, the query is sent to a recursive resolver (e.g., ISP DNS, Google `8.8.8.8`, Cloudflare `1.1.1.1`).

## 3. Recursive Resolver Begins the Lookup

The recursive resolver performs iterative queries, moving down the DNS hierarchy until it finds the IP address.

### Step 1 — Query Root Name Server

**Command example:**

```bash
dig @198.41.0.4 www.example.com
```

* `198.41.0.4` is **A.ROOT-SERVERS.NET** (one of the 13 root servers).
* **Question:** “What is the IP of `www.example.com`?”
* **Root Answer:**
  “I don’t know the IP, but `.com` domains are handled by these TLD name servers:”

  ```
  a.gtld-servers.net.  192.5.6.30
  b.gtld-servers.net.  192.33.14.30
  ...
  ```


### Step 2 — Query the TLD Name Server (.com)

**Command example:**

```bash
dig @192.5.6.30 www.example.com
```

* `192.5.6.30` is **A.GTLD-SERVERS.NET** (one of `.com`’s TLD servers).
* **Question:** “What is the IP of `www.example.com`?”
* **TLD Answer:**
  “I don’t know the IP, but `example.com` is managed by these authoritative name servers:”

  ```
  a.iana-servers.net.   199.43.135.53
  b.iana-servers.net.   199.43.133.53
  ```

### Step 3 — Query the Authoritative Name Server

**Command example:**

```bash
dig @199.43.135.53 www.example.com
```

* `199.43.135.53` is **A.IANA-SERVERS.NET** (authoritative for `example.com`).
* **Question:** “What is the IP of `www.example.com`?”
* **Authoritative Answer:**

  ```
  www.example.com. 86400 IN A 93.184.216.34
  ```

## 4. Recursive Resolver Caches the Answer

* Stores the result for the record’s TTL (e.g., 86400 seconds = 1 day).
* Returns the IP to your OS.


## 5. OS Passes IP to Browser

* Browser connects to `93.184.216.34` via TCP/UDP depending on protocol (HTTP → TCP, DNS over HTTPS → TLS).

## **6. Browser Loads the Website**

* DNS part is done — now it’s regular HTTP(S) traffic.

###  Real Flow Recap

```
Browser → OS cache → Recursive Resolver
Recursive Resolver → Root Server (.com info)
Recursive Resolver → .com TLD Server (example.com NS info)
Recursive Resolver → Authoritative NS (IP info)
Recursive Resolver → OS → Browser
Browser → Connect to IP
```

### Why This Works Like a Chain of Referrals
- Each step narrows down the search:
   - Root: Finds the right TLD.
   - TLD: Finds the right authoritative server.
   - Authoritative: Gives the actual IP.



# Types of DNS Queries

When a client asks a DNS resolver for the IP address of a domain, there are **three main query types** depending on how the resolver behaves and what information it already has.


## 1. Recursive Query

In a **recursive query**, the client (e.g., your computer’s DNS resolver) demands a complete answer from the DNS server it’s querying.

* The server must either:
  * Return the requested IP address, **or**
  * Return an error saying the domain does not exist.
* The client does **not** want partial information or referrals.

**Example Flow:**

1. Your computer asks the recursive resolver: *“What is the IP for [www.example.com?”](http://www.example.com?”)*
2. The resolver will:
   * Check its cache.
   * If not found, query the Root → TLD → Authoritative servers **on your behalf**.
3. It then returns the final IP address to you.

**Key Point:**

* **Recursive queries are what your computer sends to your ISP’s or public resolver.**
* This is why the “Recursive Resolver” in DNS resolution acts as the middleman.


## 2. Iterative Query

In an **iterative query**, the DNS server is **allowed to return the best answer it has** — even if it’s not the final IP address.

* If the server doesn’t know the IP, it returns a **referral** to another DNS server that might know.
* The client then contacts that referred server, and so on, until it finds the answer.

**Example Flow:**

1. Recursive resolver asks the Root server: *“What is the IP for [www.example.com?”](http://www.example.com?”)*
2. Root server replies: *“I don’t know, but here’s a `.com` TLD server you can try.”*
3. Resolver then queries the `.com` TLD server.
4. TLD server replies: *“I don’t know, but here’s the authoritative server for example.com.”*
5. Resolver finally asks the authoritative server and gets the IP.

**Key Point:**

* **Servers in the DNS hierarchy (Root, TLD, Authoritative) respond with iterative queries**.
* Only the recursive resolver continues the chain automatically for the client.

## 3. Non‑Recursive Query

In a **non‑recursive query**, the DNS server is asked to answer only using information it already has in its cache.

* No additional lookups are performed.
* If the answer is cached, it’s returned immediately.
* If not cached, the server returns an error or a “no data” response.

**Example Flow:**

1. A monitoring tool checks a resolver to see if it already knows `www.example.com`.
2. The resolver returns:

   * Cached IP (if available), or
   * “No data” if not cached.

**Key Point:**

* Useful when you **want to avoid extra DNS traffic**.
* Commonly used in **performance monitoring** and **load testing**.


## **Quick Comparison Table**

| Query Type        | Who Sends It?            | What Happens if Server Doesn’t Know? | Common Use Case          |
| ----------------- | ------------------------ | ------------------------------------ | ------------------------ |
| **Recursive**     | Your computer → Resolver | Resolver finds the answer for you    | Everyday browsing        |
| **Iterative**     | Resolver → Root/TLD/NS   | Server gives referral to next server | DNS resolution chain     |
| **Non‑Recursive** | Client → Resolver        | Returns only cached info             | Cache checks, monitoring |


# Anycasting and Related Communication Models 
Before diving into Anycast, let’s briefly understand the different IP communication models:

## Unicast: One-to-one communication
Unicast is the most common type of network communication. Every device (node) gets a unique IP address, and data is sent directly to a specific destination.

### How it works
- You want to send data to a single device.
- The packet contains that device’s unique IP address.
- The network routes the packet only to that device.

If multiple people need the same data, you must send it separately to each — not efficient.

## Broadcast: One‑to‑all communication in a network.
Broadcast means sending a packet to all devices in a network segment. It’s one sender → all receivers in the broadcast domain.

### How it works
- A special broadcast IP is used:
   - IPv4 example: 255.255.255.255 (local broadcast)
   - Or a subnet broadcast address: e.g., 192.168.1.255
- Every device on the local network receives the packet.

## Multicast: One‑to‑many Communication 
Multicast delivers data to a selected group of devices that have joined a multicast group. It’s one sender → many receivers (but not everyone).

### How it works
- Uses reserved IP ranges:
   - IPv4 multicast: 224.0.0.0 to 239.255.255.255
   - IPv6 multicast: ff00::/8
- Devices “subscribe” to a multicast group.
- Only subscribers receive the packets.

## Anycast: one‑to‑nearest communication
Anycast is a network routing technique where multiple servers in different locations share the same IP address. When a client sends a request to that IP, the Internet’s routing system (BGP) automatically directs the traffic to the nearest or best‑performing server. If one server becomes unavailable, traffic is rerouted to another available server without any change required on the client side.

Anycast is widely used in DNS (root servers, public resolvers, authoritative servers) and CDNs to reduce latency, increase availability, and naturally distribute load.

### Why we need anycast
Most of the Internet works via a routing scheme called Unicast. Home and office networks use Unicast; when a computer is connected to a wireless network and gets a message saying the IP address is already in use, an IP address conflict has occurred because another computer on the same Unicast network is already using the same IP. In most cases, that isn't allowed.

When a CDN node or DNS server is using a Unicast address, traffic is routed directly to the specific node. This creates a vulnerability when the network experiences extraordinary traffic such as during a DDoS attack. Because the traffic is routed directly to a particular data center, the location or its surrounding infrastructure may become overwhelmed with traffic, potentially resulting in denial-of-service to legitimate requests. That's why anycast is used instead in such cases.

## How does Anycast work?
- Multiple locations (e.g., New York, London, Tokyo) run the same service and advertise the same IP address via BGP.
- When you send a request to that IP, the Internet's routing system automatically selects the nearest/fastest server.
- If one server goes down, BGP automatically reroutes traffic to another available server.

## Anycast in DNS
DNS systems heavily use Anycast for:

1. Root Servers
   - There are 13 logical root servers (A–M).
   - In reality, there are hundreds of physical instances around the world.
   - Anycast makes all instances of "A.ROOT-SERVERS.NET" reachable at the same IP.

2. Public DNS Providers
   - Google (8.8.8.8), Cloudflare (1.1.1.1), Quad9 (9.9.9.9) are all Anycasted.
   - This ensures fast DNS lookups regardless of where you are.

3. Authoritative DNS Providers
   - Providers like Cloudflare, AWS Route 53, and Akamai Anycast their authoritative name servers.
   - Reduces query latency for websites globally.

## Benefits of Anycast in DNS
| Benefit               | Explanation                                                   |
| --------------------- | ------------------------------------------------------------- |
| **Lower Latency**     | You always hit the nearest DNS node.                          |
| **Load Distribution** | Requests are naturally spread across multiple nodes.          |
| **Resilience**        | If one node fails, traffic is automatically routed elsewhere. |
| **DDoS Mitigation**   | Large attacks are spread over many global servers.            |



## How does an Anycast network mitigate a DDoS attack?
Here’s how **Anycast** helps mitigate a DDoS attack:

---

## **How Anycast Mitigates a DDoS Attack**

1. **Traffic is Distributed Across Many Locations**

   * In Anycast, the same IP address is announced from multiple data centers worldwide.
   * When a DDoS attack targets that IP, the attack traffic is **automatically split** across all those locations instead of concentrating on a single server.

2. **Attack Load is Localized**

   * Because each attacker is routed to the **nearest** Anycast node (based on network routing), no single location gets the full volume of the attack.
   * Example: Attackers in Asia will hit Asian data centers, attackers in Europe will hit European data centers.

3. **Overwhelmed Nodes Can Withdraw from Routing**

   * If one data center becomes overloaded, it can **stop advertising** the Anycast IP via BGP.
   * Traffic is then re‑routed to other available data centers.

4. **Legitimate Traffic Can Still Get Through**

   * Even during a massive attack, users who are near healthy Anycast nodes can still reach the service.
   * This improves availability compared to Unicast, where one overloaded location could take the whole service down.


### **Visual Example**

Imagine 3 Anycast locations:

* **New York**
* **London**
* **Tokyo**

If 90 million attack packets come in:

* New York: Handles \~30M
* London: Handles \~30M
* Tokyo: Handles \~30M

Instead of **one server** choking on all 90M, **each handles a fraction**.
If New York fails, traffic from that region gets rerouted to London or Tokyo.


### Note: Anycast reduces the blast radius of a DDoS but doesn’t make you invulnerable.

### Some situations when Anycast won't save you from DDoS

1. Anycast can spread the load across locations, but each location still has a finite amount of bandwidth to the internet.
   - Example: Your London data center has a 10 Gbps connection. If attackers send 20 Gbps worth of packets from sources near London → That link is saturated, users in that region lose service.
2. When the Attack Is Sourced from a Single Geographic Region.
   - If most of the attacking machines are in one geographic area, then most of the attack traffic will hit only one Anycast node.
   - That node might still get overwhelmed, and users in that region might see downtime. 
3. If the Attack Targets Your Origin or Backend
   - If the Anycast node has to fetch content from a single origin server, and the attacker floods the origin, you can still go down.
   - Example: Your CDN edge servers are Anycasted, but your origin web server is a single Unicast IP — attackers flood the origin directly.



