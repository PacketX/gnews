# Performance
## T20/F2T12/F4T4
### Max \<find\> count in \<filter\>
- IPv4 address: 10,000,000

### Max IPv4 flow table size 32,000,000
flowCacheBaseSize: 2000000 (2000000x16=32,000,000)

### Max IPv6 flow table size 4,000,000
flowv6TableSize: 4000000

### Throughupt
- in Pps: 4,500,000 (64 bytes) (with flow table)
- in Pps: 32,000,000? (64 bytes) (without flow table)
- in/out Pps: 23,000,000 (64 bytes) (without flow table)

### Netflow eps
- throughput:   17Gbps
- uptime:  149 days = 12873600 seconds
- flow count:  545021977018
- flow per second: 42336
- flow traffic: 9.65Mbps

## T12S
### deduplication
- 64 bytes 4.3Gbps
### Throughupt
- in Pps: 12,000,000 (64 bytes) (without flow table)
- in/out Pps: 8,000,000 (64 bytes) (without flow table)

## G8/G8S
### Max \<find\> count in \<filter\>
- IPv4 address: 2000000
### Throughupt
- in Pps: 370,000 (with flow table)
- in Pps: 500,000 (without flow table)
