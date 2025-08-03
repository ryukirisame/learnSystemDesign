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
12. How DNS works with CDN and load balancing
13. Difference between www and mail. is www default?
14. Anycast IPs


# What is DNS?
Every single computer (or server) on the internet is identified by their IP address (e.g., `142.250.182.110` for Google). When one computer wants to communicate with another, it uses the destination’s IP address to connect to it. 

However, remembering numeric IP addresses is difficult for humans. We prefer to use names like `www.google.com` instead. This is where the Domain Name System (DNS) comes in. DNS translates human‑friendly domain names into machine‑readable IP addresses. And because of this, its often called _the phonebook of the internet_.

## How this "phonebook" is stored?
Since the Internet is so huge today, a central directory system cannot hold all the mapping. In addition, if the central computer fails, the whole communication network will collapse. A better solution is to distribute the information among many computers in the world. In this method, the host that needs mapping can contact the closest computer holding the needed information. This method is used by the Domain Name
System (DNS).

# Domain Name Space
It's a complete set of possible domain names. A name space that maps each IP address to a unique name can be organized in two ways: flat or hierarchical.

## Flat Name Space
In a flat name space, a name is assigned to an address. A name in this space is a sequence of characters without structure. The names may or may not have a common section; if they do, it has no meaning. 

The main disadvantage of a flat name space is that it cannot be used in a large system such as the Internet because it must be centrally controlled to avoid ambiguity and duplication. Also, whenever we have a scenario when something is being controlled "centrally" then we risk availability. 

## Hierarchical Name Space
In a hierarchical name space, each name is made of several parts. The parts are organized in an inverted-tree structure with the root at the top. The tree can have only 128 levels: level 0 (root) to level 127.

<img width="603" height="228" alt="image" src="https://github.com/user-attachments/assets/15ffd2b7-87fc-4050-82f3-2577ac8b0f20" />

Each node in the tree has a `label`, which is a string with a maximum of 63 characters. The root label is a null string (empty string). DNS requires that children of a node (nodes that branch from the same node) have different labels, which guarantees the uniqueness of the domain names.

Each node in the tree has a `domain name`. A full domain name is a sequence of labels separated by dots (.). The domain names are always read from the node up to the root. The last label is the label of the root(null). This means that a full domain name always ends in a null label, which means the last character is a dot because the null string is nothing.

<img width="545" height="349" alt="image" src="https://github.com/user-attachments/assets/1ab54393-5486-434a-9bd2-60c1e580d2e1" />

A `domain` is a subtree of the domain name space. The name of the domain is the name of the node at the top of the subtree. Note that a domain may itself be divided into domains. 

<img width="588" height="269" alt="image" src="https://github.com/user-attachments/assets/b83cf62e-ec0c-4f6d-b5c5-10c0e80db33c" />

**Fully Qualified Domain Name**: If a domain name is terminated by a null string, it is called a fully qualified domain name (FQDN). The name must end with a null label, but because null means nothing, the label ends with a dot.
**Partially Qualified Domain Name**: If a label is not terminated by a null string, it is called a partially qualified domain name (PQDN). A PQDN starts from a node, but it does not reach the root. It is used when the name to be resolved belongs to the same site as the client. Here the resolver can supply the missing part, called the suffix, to create an FQDN. 



