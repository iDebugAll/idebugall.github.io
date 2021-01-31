---
layout: post
title: Implementing Offline traceroute Tool Using Python
---

This post demonstrates an interesting use-case of radix tries usage for Longest Prefix Match.

The post was born from a question asked by an IT forum member. The summary of the question looked as follows: 

- There is a set of text files containing routing tables collected from various network devices.
- Each file represents one device.
- Device platforms and routing table formats may vary.
- It is required to analyze a routing path from any device to an arbitrary subnet or host on-demand.
- Resulting output should contain a list of routing table entries that are used for the routing to the given destination on each hop.

The one who asked a question worked as a TAC engineer. It is often that they collect or receive from the customers some text 'snapshots' of the network state for further offline analysis while troubleshooting the issues. Some automation could really save a lot of time.

I found this task interesting and also applicable to my own needs, so I decided to write a Proof-of-Concept implementation in Python 3 for Cisco IOS, IOS-XE, and ASA routing table format. 

In this article, I’ll try to reconstruct the resulting script development process and my considerations behind each step.

<cut/>

## Disclaimer
```
This is a translation of my original article in Russian I posted in June 2018.
If you are willing to help to improve the translation, please DM me.
All listed code is published under MIT license and does not provide guarantees of 
any kind.

The solution is written in Python 3.7.
An understanding of the programming and networking basics is desirable for reading.

If you found a typo, please use Ctrl+Enter or ⌘+Enter to send it to the author.
```


# Task Decomposition and Requirements Analysis

Considering the initial task summary, I would split it into two main parts:
 - Extracting the routing tables from the text files into some representation in Python data structures.
 - Analyzing routing paths based on that pre-processed routing data. 

Such logic separation will allow us to import the routing data from different sources (e.g. APIs or SNPM) and not limiting the potential scope to text file input. 

To improve route lookup performance, it is necessary to initialize the files just once on script startup. 
The solution will support Cisco IOS, IOS-XE, and ASA routing table output format for IPv4. Extensibility logic applies here as well. 

As we know, a route selection during routing table lookup relies on a Longest Prefix Match (LPM) logic. Unlike with Access Control lists, it is not enough to pick the first match. We have to find the most specific one.

Fortunately, there are fast algorithms and approaches for LPM calculation. One of them is building a so-called prefix tree (prefix trees may also be referenced to as Subnet Tries, Patricia Tries, or Radix Trees in the general case) based on a routing table.

In prefix trees, the lookup speed does not depend on the tree size (the number of prefixes), it only depends on a tree depth (maximum prefix length) which is a maximum of 32 for IPv4 (programmers would say that the search time complexity is O(k), where *k* is the maximum prefix length). In other words, route lookup using prefix trees will work with the same average speed on routing tables containing 500 and 500,000 routes.

Initial tree building time does linearly depend on a number of prefixes and their length (programmers would say that the build time complexity is O(n\*k) where *n* is the number of prefixes and *k* is the maximum prefix length), but we do this just once. Any subsequent search request will work super fast.
```
A detailed explanation of this algorithm deserves a dedicated article.
Please let me know if you are interested.
```
As we are not limited in usage of an external dependencies, we can use some existing Python library implementing this. Some of them are:

 - [SubnetTree](https://pypi.org/project/pysubnettree/). I wrote my original solution based on this library. It works really fast. But its internal code is written in C++ which causes some compatibility issues and dependency hell across operating systems. 
 - [PyTricia](https://github.com/jsommers/pytricia). This library looks best in terms of performance and compatibility in early 2021. It is written in C, so it should work more seamlessly in different OSs. So I migrated my solution to this library. Thanks to the selected code design, it was a matter of a few changed lines of code. 

We should also keep in mind that the analyzed network segment may contain routing loops. They should be detected. It should not break the script execution.<br/>
The routers may also have no route to the destination. That’s another point to note. 

Routing tables for [VRF](https://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/Network_Virtualization/PathIsol.html#wp80043)s, if present, should be saved into dedicated text files as each VRF instance represents a separate logical device from the routing and topology perspective.

Hardware performance limitations for script execution, the potential size of the routing tables and the network segments were not in a list of initial requirements. However, let’s take them into account. 
Most modern network platforms support over 1M routing entries on average. IPv4 BGP Full View size is around 814,000 prefixes as of January 2021.

My rough estimations and tests showed that in-memory processing of each 1M prefixes requires ~500MB of RAM for this scenario. Even a mid-level laptop with 8GB RAM should allow you to process a topology consisting of 17-18 routers with Full View on each of them (~12-13M prefixes in total). I believe this is enough for most of the cases. For larger network segments, the analysis can either be split into smaller scopes or moved into an external out-of-memory database.<br/>
~~640 kB ought to be enough for anybody.~~ Let's stick on an in-memory processing option.


## Parsing Source Files and Selecting Data Structures

Let's store all our text files with routing tables in a separate variable-defined sub-directory:
```python
RT_DIRECTORY = "./routing_tables"
```

Here is a reference of IOS and IOS-XE routing table output format:

*show ip route*

```
S* 0.0.0.0/0 [1/0] via 10.220.88.1
10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C 10.220.88.0/24 is directly connected, FastEthernet4
L 10.220.88.20/32 is directly connected, FastEthernet4
     1.0.0.0/32 is subnetted, 1 subnets
S       1.1.1.1 [1/0] via 212.0.0.1
                [1/0] via 192.168.0.1
D EX     10.1.198.0/24 [170/1683712] via 172.16.209.47, 1w2d, Vlan910
                       [170/1683712] via 172.16.60.33, 1w2d, Vlan60
                       [170/1683712] via 10.25.20.132, 1w2d, Vlan220
                       [170/1683712] via 10.25.20.9, 1w2d, Vlan20
     4.0.0.0/16 is subnetted, 1 subnets
O E2    4.4.0.0 [110/20] via 194.0.0.2, 00:02:00, FastEthernet0/0
     5.0.0.0/24 is subnetted, 1 subnets
D EX    5.5.5.0 [170/2297856] via 10.0.1.2, 00:12:01, Serial0/0
     6.0.0.0/16 is subnetted, 1 subnets
B       6.6.0.0 [200/0] via 195.0.0.1, 00:00:04
     172.16.0.0/26 is subnetted, 1 subnets
i L2    172.16.1.0 [115/10] via 10.0.1.2, Serial0/0
     172.20.0.0/32 is subnetted, 3 subnets
O       172.20.1.1 [110/11] via 194.0.0.2, 00:05:45, FastEthernet0/0
O       172.20.3.1 [110/11] via 194.0.0.2, 00:05:45, FastEthernet0/0
O       172.20.2.1 [110/11] via 194.0.0.2, 00:05:45, FastEthernet0/0
     10.0.0.0/8 is variably subnetted, 5 subnets, 3 masks
C       10.0.1.0/24 is directly connected, Serial0/0
D       10.0.5.0/26 [90/2297856] via 10.0.1.2, 00:12:03, Serial0/0
D       10.0.5.64/26 [90/2297856] via 10.0.1.2, 00:12:03, Serial0/0
D       10.0.5.128/26 [90/2297856] via 10.0.1.2, 00:12:03, Serial0/0
D       10.0.5.192/27 [90/2297856] via 10.0.1.2, 00:12:03, Serial0/0
     192.168.0.0/32 is subnetted, 1 subnets
D       192.168.0.1 [90/2297856] via 10.0.1.2, 00:12:03, Serial0/0
O IA 195.0.0.0/24 [110/11] via 194.0.0.2, 00:05:45, FastEthernet0/0
O E2 212.0.0.0/8 [110/20] via 194.0.0.2, 00:05:35, FastEthernet0/0
C    194.0.0.0/16 is directly connected, FastEthernet0/0
```

<br/>

Cisco ASA looks very similar. The difference is ASA displays full subnet masks instead of prefix lengths:

*show route*

```
S    10.1.1.0 255.255.255.0 [3/0] via 10.86.194.1, outside
C    10.86.194.0 255.255.254.0 is directly connected, outside
S*   0.0.0.0 0.0.0.0 [1/0] via 10.86.194.1, outside
```

</details>

<br/>

The examples show that, despite the multitude of options, all routing table entries have a predictable format. So they can be processed with regular expressions.<br/>
There are two common groups based on route entry format: *Local+Connected* types and all the rest.

The existence of multi-line routes for multi-path routing cases makes them harder to extract. We can not use simple line iteration through the content of the files because of that. One of the workarounds is to iterate through regular expression matches covering multiple lines.<br/>

Let's write such regular expressions:
```python
# Local and Connected route strings matching.
REGEXP_ROUTE_LOCAL_CONNECTED = re.compile(
    r'^(?P<routeType>[L|C])\s+'
    + r'((?P<ipaddress>\d\d?\d?\.\d\d?\d?\.\d\d?\d?\.\d\d?\d?)'
    + r'\s?'
    + r'(?P<maskOrPrefixLength>(\/\d\d?)?'
    + r'|(\d\d?\d?\.\d\d?\d?\.\d\d?\d?\.\d\d?\d?)?))'
    + r'\ is\ directly\ connected\,\ '
    + r'(?P<interface>\S+)',
    re.MULTILINE
)

# Static and dynamic route strings matching.
REGEXP_ROUTE = re.compile(
    r'^(\S\S?\*?\s?\S?\S?)'
    + r'\s+'
    + r'((?P<subnet>\d\d?\d?\.\d\d?\d?\.\d\d?\d?\.\d\d?\d?)'
    + r'\s?'
    + r'(?P<maskOrPrefixLength>(\/\d\d?)?'
    + r'|(\d\d?\d?\.\d\d?\d?\.\d\d?\d?\.\d\d?\d?)?))'
    + r'\s*'
    + r'(?P<viaPortion>(?:\n?\s+(\[\d\d?\d?\/\d+\])\s+'
    + r'via\s+(\d\d?\d?\.\d\d?\d?\.\d\d?\d?\.\d\d?\d?)(.*)\n?)+)',
    re.MULTILINE
)
```

Both regular expressions contain ([named groups](https://docs.python.org/2/library/re.html)) to make them more readable and maintainable.
You can reference the named group value within the regular expression match by its key: *subnet*/*interface*/*maskOrPrefixLength* for the prefix info and *viaPortion*/*interface* for the route destination info in our case.

The regular expression covers subnet mask and prefix length representations at once. It can be extracted by *maskOrPrefixLength* key. For a further processing, let's bring it to a common format of the prefix length as it is shorter:
```python
def convert_netmask_to_prefix_length(mask_or_pref):
    """
    Gets subnet_mask (XXX.XXX.XXX.XXX) of /prefix_length (/XX).
    For subnet_mask, converts it to /prefix_length and returns the result.
    For /prefix_length, returns as is.
    For empty input, returns "" string.
    """
    if not mask_or_pref:
        return ""
    if re.match(r"^\/\d\d?$", mask_or_pref):
        return mask_or_pref
    if re.match(r"^\d\d?\d?\.\d\d?\d?\.\d\d?\d?\.\d\d?\d?$",
                mask_or_pref):
        return (
            "/"
            + str(sum([bin(int(x)).count("1") for x in mask_or_pref.split(".")]))
        )
    return ""
```

Let's also write a regular expression for next-hop extraction from the *viaPortion* group and a regular expression for IPv4 address format check in a file and user input:
```python
# Route string VIA portion matching.
REGEXP_VIA_PORTION = re.compile(
    r'.*via\s+(\d\d?\d?\.\d\d?\d?\.\d\d?\d?\.\d\d?\d?).*'
)
# RegEx template string for IPv4 address matching.
REGEXP_IPv4_STR = (
    r'((25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.'
    + r'(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.'
    + r'(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.'
    + r'(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?))'
)
# IPv4 CIDR notation matching in user input.
REGEXP_INPUT_IPv4 = re.compile(r"^" + REGEXP_IPv4_STR + r"(\/\d\d?)?$")
```

Now let's translate our network representation into Python data structures.
All the prefixes we extract from the routing tables will be used as prefix tree keys. Each prefix tree object will be inherited from the *PyTricia* module. Search result on a prefix tree will return a list of available next-hops and a full-text representation of the matched routing entry. Another list will store a list of local interfaces with their IP-addresses for each router.
Each router will be represented by a dictionary object containing all above.
```python
# Example data structures
route_tree = pytricia.PyTricia()
route_tree[’subnet’] = ((next_hop_1, next_hop_n), raw_route_string)
interface_list = ((interface_1, ip_address_1), (interface_n, ip_address_n))
connected_networks = ((interface_1, subnet_1), (interface_n, subnet_n))
router = {
    ‘routing_table’: route_tree,
    ‘interface_list’: interface_list,
    ‘connected_networks’: connected_networks,
}
```
Now we can implement a route lookup function:
```python
def route_lookup(destination, router):
    if destination in router['routing_table']:
        return router['routing_table'][destination]
    else:
        return (None, None)
```

To distinguish between the routers, it is important to assign some unique router identifier (RID) for each of them. Router ID generation and selection algorithms might be different. In our case, let's use a filename as a RID for simplicity.<br/>
Let's put all resulting routers into a dictionary with RIDs as keys and corresponding router objects as values:
```python
ROUTERS = {
    ‘router_id_1’: router_1,
    ‘router_id_n’: router_n,
}
```

We also need to implement some next-hop RID resolution mechanism by known next-hop IP-address (think of ARP).
Let's create one more prefix tree containing IP-addresses of all discovered router as keys and RID with interface types as corresponding values:
```python
# Example
GLOBAL_INTERFACE_TREE = pytricia.PyTricia()
GLOBAL_INTERFACE_TREE[‘ip_address’] = (router_id, interface_type)

# Returns RouterID by Interface IP address which it belongs to.
def get_rid_by_interface_ip(interface_ip):
    if interface_ip in GLOBAL_INTERFACE_TREE:
        return GLOBAL_INTERFACE_TREE[interface_ip][0]
```

Now let's combine our IOS/IOS-XE/ASA format parsers into a single function. It will take a text routing table as an input and return a router dictionary object of a format we discussed earlier:
```python
def parse_show_ip_route_ios_like(raw_routing_table):
    """
    Parser for routing table text output.
    Compatible with both Cisco IOS(IOS-XE) 'show ip route'
    and Cisco ASA 'show route' output format.
    Processes input text file and write into Python data structures.
    Builds internal PyTricia search tree in 'route_tree'.
    Generates local interface list for a router in 'interface_list'
    Returns 'router' dictionary object with parsed data.
    """
    router = {}
    route_tree = pytricia.PyTricia()
    interface_list = []
    # Parse Local and Connected route strings in text.
    for raw_route_string in REGEXP_ROUTE_LOCAL_CONNECTED.finditer(raw_routing_table):
        subnet = (
            raw_route_string.group('ipaddress')
            + convert_netmask_to_prefix_length(
                raw_route_string.group('maskOrPrefixLength')
            )
        )
        interface = raw_route_string.group('interface')
        route_tree[subnet] = ((interface,), raw_route_string.group(0))
        if raw_route_string.group('routeType') == 'L':
            interface_list.append((interface, subnet,))
    if not interface_list:
        print('Failed to find routing table entries in given output')
        return None
    # parse static and dynamic route strings in text
    for raw_route_string in REGEXP_ROUTE.finditer(raw_routing_table):
        subnet = (
            raw_route_string.group('subnet')
            + convert_netmask_to_prefix_length(
                raw_route_string.group('maskOrPrefixLength')
            )
        )
        via_portion = raw_route_string.group('viaPortion')
        next_hops = []
        if via_portion.count('via') > 1:
            for line in via_portion.splitlines():
                if line:
                    next_hops.append(REGEXP_VIA_PORTION.match(line).group(1))
        else:
            next_hops.append(REGEXP_VIA_PORTION.match(via_portion).group(1))
        route_tree[subnet] = (next_hops, raw_route_string.group(0))
    router = {
        'routing_table': route_tree,
        'interface_list': interface_list,
    }
    return router
```
To improve extensibility, let's wrap all parsers into another function:
```python
def parse_text_routing_table(raw_routing_table):
    """
    Parser functions wrapper.
    Add additional parsers for alternative routing table syntaxes here.
    """
    router = parse_show_ip_route_ios_like(raw_routing_table)
    if router:
        return router
```

Finally, we need a function to go through a directory containing our routing table text files.
It will take a directory path as an input and return a dictionary of all discovered routers:
```python
def do_parse_directory(rt_directory):
    """
    Go through the specified directory and parse all .txt files.
    Generate router objects based on parse result if any.
    Populate new_routers with those router objects.
    The default key for each router object is FILENAME.
    Return new_routers.
    """
    new_routers = {}
    if not os.path.isdir(rt_directory):
        print(
            "{} directory does not exist.".format(rt_directory)
            + "Check rt_directory variable value."
        )
        return None
    start_time = time()
    print("Initializing files...")
    for FILENAME in os.listdir(rt_directory):
        if FILENAME.endswith('.txt'):
            file_init_start_time = time()
            with open(os.path.join(rt_directory, FILENAME), 'r') as f:
                print('Opening {}'.format(FILENAME))
                raw_table = f.read()
                new_router = parse_text_routing_table(raw_table)
                router_id = FILENAME.replace('.txt', '')
                if new_router:
                    new_routers[router_id] = new_router
                    if new_router['interface_list']:
                        for iface, addr in new_router['interface_list']:
                            GLOBAL_INTERFACE_TREE[addr] = (router_id, iface,)
                else:
                    print('Failed to parse ' + FILENAME)
            print(
                FILENAME
                + " parsing has been completed in {} sec".format(
                    "{:.3f}".format(time() - file_init_start_time)
                )
            )
    else:
        if not new_routers:
            print(
                "Could not find any valid .txt files with routing tables"
                + " in {} directory".format(rt_directory)
            )
        else:
            print(
                "\nAll files have been initialized"
                + " in {} sec".format("{:.3f}".format(time() - start_time))
            )
            return new_routers
```
Once we have the structured data, we can move to the routing paths analysis part of the task.


## Analyzing Routing Paths

In general, the task at this stage is to analyze the network graph. Routers are graph vertices and L3-links are graph edges. *ROUTERS* dictionary stores Router IDs as keys and next-hop IP-addresses as values. *GLOBAL_INTERFACE_TREE* returns RIDs by next-hop IP-addresses at the same time. So *ROUTERS* and *GLOBAL_INTERFACE_TREE* together define a graph adjacency table.

If we draw parallels with real routers, to find a path, you need to reproduce their high-level work logic (not taking RIB/FIB/ASIC and different optimizations into account) during the packet processing: from routing table lookup to ARP-request (*router_id* in our case) and further packet forwarding or drop depending on the result.

To achieve this, let's implement a recursive path search algorithm. Each path segment will be represented by a list containing *router_id* (RID) and *raw_route_string* (matched route string). The current path will be stored in a *path* tuple. As we might have multiple paths, the resulting list of them will be stored in a *paths* tuple. Individual *path* will be appended to *paths* once the current path analysis reached the end (the destination or no route to the destination at some point) or on routing loop detection. The function will take a RID we start from and a target IP we are searching path to as an input and return resulting *paths*.

```python
def trace_route(source_router_id, target_ip, path=[]):
    """
    Performs recursive path search from source Router ID (RID) to target subnet.
    Returns tuple of path tuples.
    Each path tuple contains a sequence of Router IDs with matched route strings.
    Multiple paths are supported.
    """
    if not source_router_id:
        return [path + [(None, None)]]
    current_router = ROUTERS[source_router_id]
    next_hop, raw_route_string = route_lookup(target_ip, current_router)
    path = path + [(source_router_id, raw_route_string)]
    paths = []
    if next_hop:
        if nexthop_is_local(next_hop[0]):
            return [path]
        for nh in next_hop:
            next_hop_rid = get_rid_by_interface_ip(nh)
            if next_hop_rid not in [r[0] for r in path]:
                inner_path = trace_route(next_hop_rid, target_ip, path)
                for p in inner_path:
                    paths.append(p)
            else:
                path = path + [(next_hop_rid+"<<LOOP DETECTED", None)]
                return [path]
    else:
        return [path]
    return paths


def nexthop_is_local(next_hop):
    """
    Check if next-hop points to the local interface.
    Will be True for Connected and Local route strings on Cisco devices.
    """
    interface_types = (
        'Eth', 'Fast', 'Gig', 'Ten', 'Port',
        'Serial', 'Vlan', 'Tunn', 'Loop', 'Null'
    )
    for type in interface_types:
        if next_hop.startswith(type):
            return True
```

Let's also implement a function to provide interactive path lookup ability to our script user.
It will perform a path search to the given IP-address from all the discovered routers:
```python
def do_user_interactive_search():
    """
    Provides interactive search dialog for users.
    Asks user for target subnet or host in CIDR notation.
    Validates input. Prints error and goes back to start for invalid input.
    Executes path search to given target from each router in global ROUTERS.
    Prints formatted path search results.
    Goes back to start.
    """
    while True:
        print('\n')
        target_subnet = input('Enter Target Subnet or Host: ')
        if not target_subnet:
            continue
        if not REGEXP_INPUT_IPv4.match(target_subnet.replace(' ', '')):
            print("incorrect input")
            continue
        lookup_start_time = time()
        for rtr in ROUTERS.keys():
            subsearch_start_time = time()
            result = trace_route(rtr, target_subnet)
            if result:
                print("\n")
                print("PATHS TO {} FROM {}".format(target_subnet, rtr))
                n = 1
                print('Detailed info:')
                for r in result:
                    print("Path {}:".format(n))
                    print([h[0] for h in r])
                    for hop in r:
                        print("ROUTER: {}".format(hop[0]))
                        print("Matched route string: \n{}".format(hop[1]))
                    else:
                        print('\n')
                    n += 1
                else:
                    print(
                        "Path search on {} has been completed in {} sec".format(
                           rtr, "{:.3f}".format(time() - subsearch_start_time)
                        )
                    )
        else:
            print(
                "\nFull search has been completed in {} sec".format(
                   "{:.3f}".format(time() - lookup_start_time),
                )
            )
```

Bringing the logic together:
```python
def main():
    global ROUTERS
    ROUTERS = do_parse_directory(RT_DIRECTORY)
    if ROUTERS:
        do_user_interactive_search()

if __name__ == "__main__":
    main() 
```

And here is a [complete solution](https://github.com/iDebugAll/toolz/blob/master/traceroute_by_routing_tables/traceroute_by_routing_tables.py)!

<details>
 <summary>The Code.</summary>
 
<script src="https://gist.github.com/iDebugAll/feef8e4339c66a92bb15ec1f87165075.js"></script>

</details>

How do we not it is working? Of course, let's do some testing. 

## Testing

I used a small topology consisting of four Cisco [CSR-1000v](https://www.cisco.com/c/en/us/products/collateral/routers/cloud-services-router-1000v-series/datasheet-c78-733443.html) routers for testing:
<img src="https://habrastorage.org/webt/qi/ga/qn/qigaqn1qe-wztcio2uqg9jtbmpe.jpeg" />

They are interconnected with GigabitEthernet 2 and 3 interfaces. All adjacent routers are EIGRP neighbors. All Connected networks are being advertised, including Loopback addresses behind each router. Besides, csr1000v-01 and csr1000v-04 have a pair of GRE tunnels between them. Both of them have a static route for 10.0.0.0/8 subnet pointing to remote GRE tunnel IP forming a routing loop.

<details>
<summary>csr1000v-01#show ip route</summary>

```
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR

Gateway of last resort is not set

S     10.0.0.0/8 [1/0] via 192.168.142.2
                 [1/0] via 192.168.141.2
      172.16.0.0/16 is variably subnetted, 2 subnets, 2 masks
C        172.16.114.0/24 is directly connected, GigabitEthernet2
L        172.16.114.5/32 is directly connected, GigabitEthernet2
      192.168.2.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.2.0/24 is directly connected, GigabitEthernet1
L        192.168.2.201/32 is directly connected, GigabitEthernet1
      192.168.12.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.12.0/24 is directly connected, GigabitEthernet2
L        192.168.12.201/32 is directly connected, GigabitEthernet2
      192.168.13.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.13.0/24 is directly connected, GigabitEthernet3
L        192.168.13.201/32 is directly connected, GigabitEthernet3
D     192.168.24.0/24 [90/3072] via 192.168.12.202, 00:06:56, GigabitEthernet2
D     192.168.34.0/24 [90/3072] via 192.168.13.203, 00:06:56, GigabitEthernet3
      192.168.141.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.141.0/30 is directly connected, Tunnel141
L        192.168.141.1/32 is directly connected, Tunnel141
      192.168.142.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.142.0/30 is directly connected, Tunnel142
L        192.168.142.1/32 is directly connected, Tunnel142
      192.168.201.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.201.0/24 is directly connected, Loopback201
L        192.168.201.201/32 is directly connected, Loopback201
D     192.168.202.0/24 
           [90/130816] via 192.168.12.202, 00:05:44, GigabitEthernet2
D     192.168.203.0/24 
           [90/130816] via 192.168.13.203, 00:06:22, GigabitEthernet3
D     192.168.204.0/24 
           [90/131072] via 192.168.13.203, 00:06:56, GigabitEthernet3
           [90/131072] via 192.168.12.202, 00:06:56, GigabitEthernet2
```

</details>

<details>
<summary>csr1000v-02#show ip route</summary>

```
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR

Gateway of last resort is not set

      192.168.2.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.2.0/24 is directly connected, GigabitEthernet1
L        192.168.2.202/32 is directly connected, GigabitEthernet1
      192.168.12.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.12.0/24 is directly connected, GigabitEthernet2
L        192.168.12.202/32 is directly connected, GigabitEthernet2
D     192.168.13.0/24 [90/3072] via 192.168.12.201, 00:46:17, GigabitEthernet2
      192.168.24.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.24.0/24 is directly connected, GigabitEthernet3
L        192.168.24.202/32 is directly connected, GigabitEthernet3
D     192.168.34.0/24 [90/3072] via 192.168.24.204, 00:46:15, GigabitEthernet3
D     192.168.201.0/24 
           [90/130816] via 192.168.12.201, 00:36:59, GigabitEthernet2
      192.168.202.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.202.0/24 is directly connected, Loopback202
L        192.168.202.202/32 is directly connected, Loopback202
D     192.168.203.0/24 
           [90/131072] via 192.168.24.204, 00:06:31, GigabitEthernet3
           [90/131072] via 192.168.12.201, 00:06:31, GigabitEthernet2
D     192.168.204.0/24 
           [90/130816] via 192.168.24.204, 00:37:26, GigabitEthernet3
```

</details>

<details>
<summary>csr1000v-03#show ip route</summary>

```
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR

Gateway of last resort is not set

      192.168.2.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.2.0/24 is directly connected, GigabitEthernet1
L        192.168.2.203/32 is directly connected, GigabitEthernet1
D     192.168.12.0/24 [90/3072] via 192.168.13.201, 00:46:12, GigabitEthernet3
      192.168.13.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.13.0/24 is directly connected, GigabitEthernet3
L        192.168.13.203/32 is directly connected, GigabitEthernet3
D     192.168.24.0/24 [90/3072] via 192.168.34.204, 00:46:12, GigabitEthernet2
      192.168.34.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.34.0/24 is directly connected, GigabitEthernet2
L        192.168.34.203/32 is directly connected, GigabitEthernet2
D     192.168.201.0/24 
           [90/130816] via 192.168.13.201, 00:36:56, GigabitEthernet3
D     192.168.202.0/24 
           [90/131072] via 192.168.34.204, 00:05:51, GigabitEthernet2
           [90/131072] via 192.168.13.201, 00:05:51, GigabitEthernet3
      192.168.203.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.203.0/24 is directly connected, Loopback203
L        192.168.203.203/32 is directly connected, Loopback203
D     192.168.204.0/24 
           [90/130816] via 192.168.34.204, 00:37:22, GigabitEthernet2
```

</details>

<details>
<summary>csr1000v-04#show ip route</summary>

```
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR

Gateway of last resort is not set

S     10.0.0.0/8 [1/0] via 192.168.142.1
                 [1/0] via 192.168.141.1
      192.168.2.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.2.0/24 is directly connected, GigabitEthernet1
L        192.168.2.204/32 is directly connected, GigabitEthernet1
D     192.168.12.0/24 [90/3072] via 192.168.24.202, 00:46:17, GigabitEthernet3
D     192.168.13.0/24 [90/3072] via 192.168.34.203, 00:46:19, GigabitEthernet2
      192.168.24.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.24.0/24 is directly connected, GigabitEthernet3
L        192.168.24.204/32 is directly connected, GigabitEthernet3
      192.168.34.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.34.0/24 is directly connected, GigabitEthernet2
L        192.168.34.204/32 is directly connected, GigabitEthernet2
      192.168.141.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.141.0/30 is directly connected, Tunnel141
L        192.168.141.2/32 is directly connected, Tunnel141
      192.168.142.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.142.0/30 is directly connected, Tunnel142
L        192.168.142.2/32 is directly connected, Tunnel142
D     192.168.201.0/24 
           [90/131072] via 192.168.34.203, 00:37:02, GigabitEthernet2
           [90/131072] via 192.168.24.202, 00:37:02, GigabitEthernet3
D     192.168.202.0/24 
           [90/130816] via 192.168.24.202, 00:05:57, GigabitEthernet3
D     192.168.203.0/24 
           [90/130816] via 192.168.34.203, 00:06:34, GigabitEthernet2
      192.168.204.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.204.0/24 is directly connected, Loopback204
L        192.168.204.204/32 is directly connected, Loopback204
```

</details>

Let's save *show ip route* files into separate files named by hostnames inside *./routing_tables/* directory.
Now we can run the script:

<details>
<summary>$ python3.7 traceroute_by_routing_tables.py</summary>

```
$ python3.7 traceroute_by_routing_tables.py
Initializing files...
Opening csr1000v-01.txt
csr1000v-01.txt parsing has been completed in 0.001 sec
Opening csr1000v-02.txt
csr1000v-02.txt parsing has been completed in 0.001 sec
Opening csr1000v-03.txt
csr1000v-03.txt parsing has been completed in 0.001 sec
Opening csr1000v-04.txt
csr1000v-04.txt parsing has been completed in 0.001 sec

All files have been initialized in 0.003 sec


Enter Target Subnet or Host:
```

</details>


All the files are processed as expected. The script expects an IP-address input to analyze paths.
Let's put several IP-addresses subsequently and compare the output with the data from our routers:

<details>
<summary>Looking up paths to 192.168.204.204 (Loopback204 on csr1000v-04)</summary>

All the routers should have paths to this destination.

<details>
<summary>Enter Target Subnet or Host: 192.168.204.204</summary>

```
Enter Target Subnet or Host: 192.168.204.204


PATHS TO 192.168.204.204 FROM csr1000v-04
Detailed info:
Path 1:
['csr1000v-04']
ROUTER: csr1000v-04
Matched route string: 
L        192.168.204.204/32 is directly connected, Loopback204


Path search on csr1000v-04 has been completed in 0.000 sec


PATHS TO 192.168.204.204 FROM csr1000v-03
Detailed info:
Path 1:
['csr1000v-03', 'csr1000v-04']
ROUTER: csr1000v-03
Matched route string: 
D     192.168.204.0/24 
           [90/130816] via 192.168.34.204, 00:37:22, GigabitEthernet2

ROUTER: csr1000v-04
Matched route string: 
L        192.168.204.204/32 is directly connected, Loopback204


Path search on csr1000v-03 has been completed in 0.000 sec


PATHS TO 192.168.204.204 FROM csr1000v-02
Detailed info:
Path 1:
['csr1000v-02', 'csr1000v-04']
ROUTER: csr1000v-02
Matched route string: 
D     192.168.204.0/24 
           [90/130816] via 192.168.24.204, 00:37:26, GigabitEthernet3
ROUTER: csr1000v-04
Matched route string: 
L        192.168.204.204/32 is directly connected, Loopback204


Path search on csr1000v-02 has been completed in 0.000 sec


PATHS TO 192.168.204.204 FROM csr1000v-01
Detailed info:
Path 1:
['csr1000v-01', 'csr1000v-03', 'csr1000v-04']
ROUTER: csr1000v-01
Matched route string: 
D     192.168.204.0/24 
           [90/131072] via 192.168.13.203, 00:06:56, GigabitEthernet3
           [90/131072] via 192.168.12.202, 00:06:56, GigabitEthernet2
ROUTER: csr1000v-03
Matched route string: 
D     192.168.204.0/24 
           [90/130816] via 192.168.34.204, 00:37:22, GigabitEthernet2

ROUTER: csr1000v-04
Matched route string: 
L        192.168.204.204/32 is directly connected, Loopback204


Path 2:
['csr1000v-01', 'csr1000v-02', 'csr1000v-04']
ROUTER: csr1000v-01
Matched route string: 
D     192.168.204.0/24 
           [90/131072] via 192.168.13.203, 00:06:56, GigabitEthernet3
           [90/131072] via 192.168.12.202, 00:06:56, GigabitEthernet2
ROUTER: csr1000v-02
Matched route string: 
D     192.168.204.0/24 
           [90/130816] via 192.168.24.204, 00:37:26, GigabitEthernet3
ROUTER: csr1000v-04
Matched route string: 
L        192.168.204.204/32 is directly connected, Loopback204


Path search on csr1000v-01 has been completed in 0.000 sec

Full search has been completed in 0.001 sec
```

</details>

The script found some paths. Now let's check the route selection right on csr1000v-01:

<details>
<summary>csr1000v-01#show ip route 192.168.204.204</summary>

```
csr1000v-01#show ip route 192.168.204.204
Routing entry for 192.168.204.0/24
  Known via "eigrp 200", distance 90, metric 131072, type internal
  Redistributing via eigrp 200
  Last update from 192.168.13.203 on GigabitEthernet3, 00:02:15 ago
  Routing Descriptor Blocks:
    192.168.13.203, from 192.168.13.203, 00:02:15 ago, via GigabitEthernet3
      Route metric is 131072, traffic share count is 1
      Total delay is 5020 microseconds, minimum bandwidth is 1000000 Kbit
      Reliability 255/255, minimum MTU 1500 bytes
      Loading 1/255, Hops 2
  * 192.168.12.202, from 192.168.12.202, 00:02:15 ago, via GigabitEthernet2
      Route metric is 131072, traffic share count is 1
      Total delay is 5020 microseconds, minimum bandwidth is 1000000 Kbit
      Reliability 255/255, minimum MTU 1500 bytes
      Loading 1/255, Hops 2
```

</details>

csr1000v-01 displays two equal-cost router learned by EIGRP through csr1000v-02 and csr1000v-03. 
The script returns both available paths: ['csr1000v-01', 'csr1000v-03', 'csr1000v-04'] and ['csr1000v-01', 'csr1000v-02', 'csr1000v-04'].<br/>

To be sure:

<details>
<summary>csr1000v-02#show ip route 192.168.204.204</summary>

```
csr1000v-02#show ip route 192.168.204.204
Routing entry for 192.168.204.0/24
  Known via "eigrp 200", distance 90, metric 130816, type internal
  Redistributing via eigrp 200
  Last update from 192.168.24.204 on GigabitEthernet3, 00:08:48 ago
  Routing Descriptor Blocks:
  * 192.168.24.204, from 192.168.24.204, 00:08:48 ago, via GigabitEthernet3
      Route metric is 130816, traffic share count is 1
      Total delay is 5010 microseconds, minimum bandwidth is 1000000 Kbit
      Reliability 255/255, minimum MTU 1500 bytes
      Loading 1/255, Hops 1
```

</details>

<details>
<summary>csr1000v-03#show ip route 192.168.204.204</summary>

```
csr1000v-3#show ip route 192.168.204.204
Routing entry for 192.168.204.0/24
  Known via "eigrp 200", distance 90, metric 130816, type internal
  Redistributing via eigrp 200
  Last update from 192.168.34.204 on GigabitEthernet2, 00:08:45 ago
  Routing Descriptor Blocks:
  * 192.168.34.204, from 192.168.34.204, 00:08:45 ago, via GigabitEthernet2
      Route metric is 130816, traffic share count is 1
      Total delay is 5010 microseconds, minimum bandwidth is 1000000 Kbit
      Reliability 255/255, minimum MTU 1500 bytes
      Loading 1/255, Hops 1
```
</details>

<details>
<summary>csr1000v-04#show ip route 192.168.204.204</summary>

```
csr1000v-04#show ip route 192.168.204.204
Routing entry for 192.168.204.204/32
  Known via "connected", distance 0, metric 0 (connected)
  Routing Descriptor Blocks:
  * directly connected, via Loopback204
      Route metric is 0, traffic share count is 1
```

</details>

Both csr1000v-02 and csr1000v-03 have a single route learned by EIGRP to csr1000v-4.
On csr1000v-04, the route leads to a Connected network on Loopback204.
The script output is correct: ['csr1000v-02', 'csr1000v-04'] from csr1000v-02, ['csr1000v-03', 'csr1000v-04'] from csr1000v-03, and a route to itself ['csr1000v-04'] from csr1000v-04.
</details>

<details>
<summary>Looking up 10.10.10.0/24 (does not exist in the topology). Also a routing loop test case.</summary>

<details>
<summary>Enter Target Subnet or Host: 10.10.10.0/24</summary>

```
Enter Target Subnet or Host: 10.10.10.0/24

PATHS TO 10.10.10.0/24 FROM csr1000v-04
Detailed info:
Path 1:
['csr1000v-04', 'csr1000v-01', 'csr1000v-04<<LOOP DETECTED']
ROUTER: csr1000v-04
Matched route string: 
S     10.0.0.0/8 [1/0] via 192.168.142.1
                 [1/0] via 192.168.141.1

ROUTER: csr1000v-01
Matched route string: 
S     10.0.0.0/8 [1/0] via 192.168.142.2
                 [1/0] via 192.168.141.2

ROUTER: csr1000v-04<<LOOP DETECTED
Matched route string: 
None


Path 2:
['csr1000v-04', 'csr1000v-01', 'csr1000v-04<<LOOP DETECTED']
ROUTER: csr1000v-04
Matched route string: 
S     10.0.0.0/8 [1/0] via 192.168.142.1
                 [1/0] via 192.168.141.1

ROUTER: csr1000v-01
Matched route string: 
S     10.0.0.0/8 [1/0] via 192.168.142.2
                 [1/0] via 192.168.141.2

ROUTER: csr1000v-04<<LOOP DETECTED
Matched route string: 
None


Path search on csr1000v-04 has been completed in 0.000 sec


PATHS TO 10.10.10.0/24 FROM csr1000v-03
Detailed info:
Path 1:
['csr1000v-03']
ROUTER: csr1000v-03
Matched route string: 
None


Path search on csr1000v-03 has been completed in 0.000 sec


PATHS TO 10.10.10.0/24 FROM csr1000v-02
Detailed info:
Path 1:
['csr1000v-02']
ROUTER: csr1000v-02
Matched route string: 
None


Path search on csr1000v-02 has been completed in 0.000 sec


PATHS TO 10.10.10.0/24 FROM csr1000v-01
Detailed info:
Path 1:
['csr1000v-01', 'csr1000v-04', 'csr1000v-01<<LOOP DETECTED']
ROUTER: csr1000v-01
Matched route string: 
S     10.0.0.0/8 [1/0] via 192.168.142.2
                 [1/0] via 192.168.141.2

ROUTER: csr1000v-04
Matched route string: 
S     10.0.0.0/8 [1/0] via 192.168.142.1
                 [1/0] via 192.168.141.1

ROUTER: csr1000v-01<<LOOP DETECTED
Matched route string: 
None


Path 2:
['csr1000v-01', 'csr1000v-04', 'csr1000v-01<<LOOP DETECTED']
ROUTER: csr1000v-01
Matched route string: 
S     10.0.0.0/8 [1/0] via 192.168.142.2
                 [1/0] via 192.168.141.2

ROUTER: csr1000v-04
Matched route string: 
S     10.0.0.0/8 [1/0] via 192.168.142.1
                 [1/0] via 192.168.141.1

ROUTER: csr1000v-01<<LOOP DETECTED
Matched route string: 
None


Path search on csr1000v-01 has been completed in 0.003 sec

Full search has been completed in 0.004 sec
```

</details>

We've got result. Let's check the routers:

<details>
<summary>csr1000v-01#show ip route 10.10.10.0 255.255.255.0</summary>

```
csr1000v-01#show ip route 10.10.10.0 255.255.255.0
Routing entry for 10.0.0.0/8
  Known via "static", distance 1, metric 0
  Routing Descriptor Blocks:
  * 192.168.142.2
      Route metric is 0, traffic share count is 1
    192.168.141.2
      Route metric is 0, traffic share count is 1
```

</details>

<details>
<summary>csr1000v-04#show ip route 10.10.10.0 255.255.255.0</summary>

```
csr1000v-04#show ip route 10.10.10.0 255.255.255.0
Routing entry for 10.0.0.0/8
  Known via "static", distance 1, metric 0
  Routing Descriptor Blocks:
    192.168.142.1
      Route metric is 0, traffic share count is 1
  * 192.168.141.1
      Route metric is 0, traffic share count is 1
```

</details>

As discussed, csr1000v-01 and csr1000v-04 have equal-cost static routes to a wide 10.0.0.0/8 network pointing to each other through the tunnel interfaces. It creates a routing loop. The script successfully detects this and shows both paths for each:

```
PATHS TO 10.10.10.0/24 FROM csr1000v-01
Path 1:
['csr1000v-01', 'csr1000v-04', 'csr1000v-01<<LOOP DETECTED']
Path 2:
['csr1000v-01', 'csr1000v-04', 'csr1000v-01<<LOOP DETECTED']

PATHS TO 10.10.10.0/24 FROM csr1000v-04
Path 1:
['csr1000v-04', 'csr1000v-01', 'csr1000v-04<<LOOP DETECTED']
Path 2:
['csr1000v-04', 'csr1000v-01', 'csr1000v-04<<LOOP DETECTED']
```

<details>
<summary>csr1000v-02#show ip route 10.10.10.0 255.255.255.0</summary>

```
csr1000v-02#show ip route 10.10.10.0 255.255.255.0
% Network not in table
```

</details>

<details>
<summary>csr1000v-3#show ip route 10.10.10.0 255.255.255.0</summary>

```
csr1000v-3#show ip route 10.10.10.0 255.255.255.0
% Network not in table
``` 

</details>

csr1000v-02 and csr1000v-03 have no route to such destination. The script shows the same result.

</details>

All common test cases are covered. The script result matches the output we get from the actual network devices using CLI.

## Conclusion

The resulting solution is not perfect but it provides great lookup speed thanks to efficient algorithms and successfully solves the original task. The source code is published on my [GitHub page](https://github.com/iDebugAll/toolz/tree/master/traceroute_by_routing_tables).

The solution has some room for improvement and adding new analysis features. Additional parsers for alternative routing table syntax can easily be added by design. IPv6 is supported by *PyTricia* library natively.

I also tested the script on a BGP Full View routing table output with 700,000+ prefixes. On my good old MacBook Pro with Intel Core i5 and 8GB RAM, an initialization time takes less than 10 seconds. Memory consumption is around 320-350MB. Once it is initialized, any lookup to a routing table of that size still takes milliseconds as expected. 

In 2021, some full-blown network analysis tools like [PyATS](https://developer.cisco.com/docs/pyats/) and [Batfish](https://www.batfish.org) could be a better starting point for more complex scenarios. However, being able to develop custom tools for such corner cases is still relevant.

Hope it helps someone solve some real issues or inspire to develop some automation on his or her own. 
Thank you for reading.
