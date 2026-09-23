# 效能
## T20/F2T12/F4T4
### 單一 \<filter\> 內 \<find\> 數量上限
- IPv4 位址: 10,000,000

### IPv4 flow table 大小上限 32,000,000
flowCacheBaseSize: 2000000 (2000000x16=32,000,000)

### IPv6 flow table 大小上限 4,000,000
flowv6TableSize: 4000000

### 吞吐量
- in Pps: 4,600,000 (64 bytes) (有 flow table)
- in Pps: 32,000,000 (64 bytes) (無 flow table)
- in/out Pps: 23,000,000 (64 bytes) (無 flow table)

### Netflow eps
- 吞吐量:   17Gbps
- 開機時間:  149 天 = 12873600 seconds
- flow 總數:  545021977018
- 每秒 flow 數: 42336
- flow 流量: 9.65Mbps

## T12S
### 去重複(deduplication)
- 64 bytes 4.3Gbps
### 吞吐量
- in Pps: 12,000,000 (64 bytes) (無 flow table)
- in/out Pps: 8,000,000 (64 bytes) (無 flow table)

## G8/G8S
### 單一 \<filter\> 內 \<find\> 數量上限
- IPv4 位址: 2000000
### 吞吐量
- in Pps: 370,000 (有 flow table)
- in Pps: 500,000 (無 flow table)

## DPDK 
```
cn103(8 cores)
in/out Pps:  4,500,000 (64 bytes) (有 flow table)
in/out Pps: 14,000,000 (64 bytes) (無 flow table)
//無法取得 input drop 統計

cn96(24 cores)
in/out Pps:  7,500,000 (64 bytes) (有 flow table)
in/out Pps:  15,000,000 (64 bytes) (無 flow table)

cn98(32 cores)
in/out Pps: 14,000,000 (64 bytes) (有 flow table)
in/out Pps: 32,000,000 (64 bytes) (無 flow table)

cn106(24 cores)
in/out Pps: 20,000,000 (64 bytes) (有 flow table)
in/out Pps: 42,000,000 (64 bytes) (無 flow table)
```
