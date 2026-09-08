# subsplit

A command line tool that takes an IP block and shows you every subnet inside it.

No install headaches, no web page, no login. You type one command and get a
clean table of every subnet with its network address, usable host range,
broadcast address, and mask.

## Why

Most subnet calculators online make you do one block at a time and copy the
results by hand. `subsplit` prints the whole set at once, works offline, and
outputs JSON so you can pipe it into whatever you are building.

## Install

```
pip install subsplit
```

Requires Python 3.9 or later. No dependencies.

## Usage

Split a block into subnets of a given prefix length:

```
subsplit 192.168.1.0/24 --prefix 26
```

Output:

```
Parent block: 192.168.1.0/24
Mask: 255.255.255.0
Total addresses: 256

Splitting into /26 (4 subnets, 62 usable hosts each)

#   Network          Usable range                      Broadcast        Mask
1   192.168.1.0/26   192.168.1.1 - 192.168.1.62        192.168.1.63     255.255.255.192
2   192.168.1.64/26  192.168.1.65 - 192.168.1.126      192.168.1.127    255.255.255.192
3   192.168.1.128/26 192.168.1.129 - 192.168.1.190     192.168.1.191    255.255.255.192
4   192.168.1.192/26 192.168.1.193 - 192.168.1.254     192.168.1.255    255.255.255.192
```

Ask for a number of subnets instead and let the tool work out the prefix:

```
subsplit 10.0.0.0/16 --subnets 8
```

Ask for a number of hosts per subnet:

```
subsplit 172.16.0.0/20 --hosts 500
```

The tool rounds up to the next prefix that fits the requirement and tells you
how many addresses are wasted.

## Options

```
--prefix N       Split into subnets of prefix length N
--subnets N      Split into at least N subnets
--hosts N        Split into subnets that hold at least N usable hosts each
--json           Output as JSON instead of a table
--csv            Output as CSV
--no-header      Print rows only, useful for piping
--limit N        Show only the first N subnets (default 256)
--bits           Show the binary breakdown of the network and host portions
```

## JSON output

```
subsplit 192.168.1.0/24 --prefix 26 --json
```

```json
{
  "parent": "192.168.1.0/24",
  "prefix": 26,
  "subnet_count": 4,
  "usable_hosts_per_subnet": 62,
  "subnets": [
    {
      "network": "192.168.1.0",
      "cidr": "192.168.1.0/26",
      "mask": "255.255.255.192",
      "first_usable": "192.168.1.1",
      "last_usable": "192.168.1.62",
      "broadcast": "192.168.1.63",
      "total_addresses": 64,
      "usable_hosts": 62
    }
  ]
}
```

## Learning mode

Pass `--bits` to see where the split actually happens:

```
subsplit 192.168.1.0/24 --prefix 26 --bits
```

```
192.168.1.0/24  ->  /26
Borrowed 2 bits from the host portion.

11000000.10101000.00000001.00 000000
                             ^^
                             borrowed bits

00 -> 192.168.1.0/26
01 -> 192.168.1.64/26
10 -> 192.168.1.128/26
11 -> 192.168.1.192/26
```

## Notes on /31 and /32

A /31 has no broadcast address and is used for point to point links, so both
addresses are treated as usable. A /32 is a single host route. The tool handles
both instead of refusing them like many calculators do.

## Roadmap

- VLSM planning, where you give a list of departments with different host counts
- Overlap and gap auditing across a set of allocated subnets
- Export to Cisco interface config and DHCP scope files
- IPv6 support

## Contributing

Pull requests welcome. Run the tests with `pytest` before opening one.

## License

MIT
