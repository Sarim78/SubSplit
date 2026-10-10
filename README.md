# subsplit

[![Tests](https://github.com/Sarim78/subsplit/actions/workflows/test.yml/badge.svg)](https://github.com/Sarim78/subsplit/actions/workflows/test.yml)
[![PyPI](https://img.shields.io/pypi/v/subsplit.svg)](https://pypi.org/project/subsplit/)
[![Python](https://img.shields.io/pypi/pyversions/subsplit.svg)](https://pypi.org/project/subsplit/)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

A command line tool that takes an IP block and shows you every subnet inside it.

No install headaches, no web page, no login. You type one command and get a clean table of every subnet with its network address, usable host range, broadcast address, and mask.

## Why

Most subnet calculators online make you do one block at a time and copy the results by hand. `subsplit` prints the whole set at once, works offline, and outputs JSON or CSV so you can pipe it into whatever you are building.

## Install

```
pip install subsplit
```

Or, to keep it isolated from your other Python packages:

```
pipx install subsplit
```

Requires Python 3.9 or later. No dependencies.

## Usage

You give `subsplit` a parent block and exactly one way to split it: `--prefix`, `--subnets`, or `--hosts`.

### Split by prefix length

```
subsplit 192.168.1.0/24 --prefix 26
```

Output:

```
Parent block: 192.168.1.0/24
Mask: 255.255.255.0
Total addresses: 256

Splitting into /26 (4 subnets, 62 usable hosts each)

#  Network           Usable range                   Broadcast      Mask
1  192.168.1.0/26    192.168.1.1 - 192.168.1.62     192.168.1.63   255.255.255.192
2  192.168.1.64/26   192.168.1.65 - 192.168.1.126   192.168.1.127  255.255.255.192
3  192.168.1.128/26  192.168.1.129 - 192.168.1.190  192.168.1.191  255.255.255.192
4  192.168.1.192/26  192.168.1.193 - 192.168.1.254  192.168.1.255  255.255.255.192
```

### Split by number of subnets

Ask for a number of subnets and let the tool work out the prefix:

```
subsplit 10.0.0.0/16 --subnets 8
```

```
Parent block: 10.0.0.0/16
Mask: 255.255.0.0
Total addresses: 65536

Requested at least 8 subnets, using /19
Splitting into /19 (8 subnets, 8190 usable hosts each)
...
```

If the number you ask for is not a power of two, the tool rounds up to the next prefix that fits and tells you how many extra subnets you got.

### Split by hosts per subnet

Ask for a number of usable hosts per subnet:

```
subsplit 172.16.0.0/20 --hosts 500
```

```
Parent block: 172.16.0.0/20
Mask: 255.255.240.0
Total addresses: 4096

Requested at least 500 usable hosts per subnet, using /23
Splitting into /23 (8 subnets, 510 usable hosts each)
Unused per subnet: 10 addresses
...
```

The tool picks the smallest subnet that still fits your host count and tells you how many addresses are left unused in each one.

## Options

```
--prefix N       Split into subnets of prefix length N
--subnets N      Split into at least N subnets
--hosts N        Split into subnets that hold at least N usable hosts each
--json           Output as JSON instead of a table
--csv            Output as CSV
--no-header      Print rows only, useful for piping (table and CSV)
--limit N        Show only the first N subnets (default 256)
--bits           Show the binary breakdown of the network and host portions
--version        Print the version and exit
-h, --help       Show help and exit
```

`--prefix`, `--subnets`, and `--hosts` cannot be combined. Pick one. The same goes for `--json` and `--csv`.

### About --limit

Large splits can produce a lot of subnets. Splitting 10.0.0.0/8 into /32 gives over 16 million. By default `subsplit` shows the first 256 and tells you how many there are in total:

```
Showing 256 of 65536 subnets. Use --limit to show more.
```

The tool generates subnets as it prints them, so even huge splits start instantly and stay light on memory.

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
  "truncated": false,
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

The example above shows only the first subnet to save space. Real output includes every subnet up to `--limit`.

`subnet_count` is always the full total. If `--limit` cut the list short, `truncated` is `true` and `subnets` holds only the subnets that were shown.

Warnings are written to stderr, never stdout, so piped JSON and CSV stay clean.

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

`--limit` applies here too, so large splits do not flood your terminal.

## Notes on /31 and /32

A /31 has no broadcast address and is used for point to point links (RFC 3021), so both addresses are treated as usable. The table shows `-` in the broadcast column and JSON sets `broadcast` to `null`.

A /32 is a single host route. It has one usable address and no broadcast.

The tool handles both instead of refusing them like many calculators do.

## Input handling

If you pass a block with host bits set, like `192.168.1.77/24`, the tool uses the network address (`192.168.1.0/24`) and prints a one line warning to stderr.

Requests that cannot fit, like splitting a /24 into /20 subnets or asking for more hosts than the parent block holds, produce a clear error that says what would fit instead.

IPv6 is not supported yet. See the roadmap.

## Exit codes

```
0   Success
1   Valid command, but the request is impossible (for example, the subnets do not fit)
2   Invalid usage (bad flags, bad input, conflicting options)
```

## Roadmap

* VLSM planning, where you give a list of departments with different host counts
* Overlap and gap auditing across a set of allocated subnets
* Export to Cisco interface config and DHCP scope files
* IPv6 support

## Contributing

Pull requests welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for setting up a dev environment. Run the tests with `pytest` and the linters with `ruff check .` and `mypy src` before opening one.

## License

MIT. See [LICENSE](LICENSE).
