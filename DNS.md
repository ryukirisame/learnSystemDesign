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
  - Root server replies: “I don’t know the IP, but .com TLD servers can help you. Here are their IPs for .com TLD servers.”
