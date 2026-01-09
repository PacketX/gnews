# Docs
## Link
- [GRISM XML](https://packetx.gitbook.io/grism-xml/)
## Manual
- [PacketX-Grism-F2T12.pptx](https://raw.githubusercontent.com/PacketX/gnews/master/docs/PacketX-Grism-F2T12.pptx)
- [PacketX-Grism-T20.pptx](https://raw.githubusercontent.com/PacketX/gnews/master/docs/PacketX-Grism-T20.pptx)
- [PacketX-Grism-T12s.pptx](https://raw.githubusercontent.com/PacketX/gnews/master/docs/PacketX-Grism-T12s.pptx)
- [PacketX-Grism-G8s(en).pptx](https://raw.githubusercontent.com/PacketX/gnews/master/docs/PacketX-Grism-G8s(en).pptx)
- [PacketX GRISM T20繁體中文操作手冊V1.4.docx](https://raw.githubusercontent.com/PacketX/gnews/master/docs/PacketX%20GRISM%20T20繁體中文操作手冊V1.4.docx)
- [20250716_PacketX_G8s_G8i_T12s_T12s-BP2_T20_Introduction_v1.4.pptx.pdf](https://raw.githubusercontent.com/PacketX/gnews/master/docs/20250716_PacketX_G8s_G8i_T12s_T12s-BP2_T20_Introduction_v1.4.pptx.pdf)
- [GRISM API更新5-tuple黑名單live阻擋.docx](https://raw.githubusercontent.com/PacketX/gnews/master/docs/GRISM%20API更新5-tuple黑名單live阻擋.docx)
- [Grism_syslog_fields.docx](https://raw.githubusercontent.com/PacketX/gnews/master/docs/Grism_syslog_fields.docx)

# Performance Test
## Files
- [PacketX GRISM中國ip過濾效能測試.docx](https://raw.githubusercontent.com/PacketX/gnews/master/docs/PacketX%20GRISM中國ip過濾效能測試.docx)

## T20/F2T12/F4T4
### Max \<find\> count in \<filter\>
- IPv4 address: 10,000,000

### Max IPv4 flow table size 32,000,000
flowCacheBaseSize: 2000000 (2000000x16=32,000,000)

### Max IPv6 flow table size 4,000,000
flowv6TableSize: 4000000

### Throughupt
- in Pps: 4,600,000 (64 bytes) (with flow table)
- in Pps: 32,000,000 (64 bytes) (without flow table)
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

## DPDK 
```
cn103(8 cores)
in/out Pps:  4,500,000 (64 bytes) (with flow table)
in/out Pps: 14,000,000 (64 bytes) (without flow table)
//can't get input drop statistics

cn96(24 cores)
in/out Pps:  7,500,000 (64 bytes) (with flow table)
in/out Pps:  15,000,000 (64 bytes) (without flow table)

cn98(32 cores)
in/out Pps: 14,000,000 (64 bytes) (with flow table)
in/out Pps: 32,000,000 (64 bytes) (without flow table)

cn106(24 cores)
in/out Pps: 20,000,000 (64 bytes) (with flow table)
in/out Pps: 42,000,000 (64 bytes) (without flow table)
```
