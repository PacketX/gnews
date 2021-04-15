# Performance
## T20/F2T12/F4T4
### Max \<find\> count in \<filter\>
- IPv4 address: 10000000

### Max IPv4 flow table size 100,000,000
flowCacheBaseSize: 6250000 (6250000x16=100,000,000)

### Max IPv6 flow table size 4,000,000
flowv6TableSize: 4000000

## G8
### Max \<find\> count in \<filter\>
- IPv4 address: 2000000


## iperf test
### G8
```
iperf -c 192.168.1.201 -u -b 1000m -t 10 -i 1 -f m 
------------------------------------------------------------
Client connecting to 192.168.1.201, UDP port 5001
Sending 1470 byte datagrams, IPG target: 11.76 us (kalman adjust)
UDP buffer size: 0.20 MByte (default)
------------------------------------------------------------
[  3] local 192.168.1.59 port 40892 connected with 192.168.1.201 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0- 1.0 sec  92.3 MBytes   774 Mbits/sec
[  3]  1.0- 2.0 sec  87.6 MBytes   735 Mbits/sec
[  3]  2.0- 3.0 sec  87.7 MBytes   736 Mbits/sec
[  3]  3.0- 4.0 sec  87.6 MBytes   735 Mbits/sec
[  3]  4.0- 5.0 sec  87.4 MBytes   733 Mbits/sec
[  3]  5.0- 6.0 sec  86.4 MBytes   725 Mbits/sec
[  3]  6.0- 7.0 sec  86.9 MBytes   729 Mbits/sec
[  3]  7.0- 8.0 sec  87.7 MBytes   736 Mbits/sec
[  3]  8.0- 9.0 sec  87.6 MBytes   735 Mbits/sec
[  3]  0.0-10.0 sec   878 MBytes   737 Mbits/sec
[  3] Sent 626340 datagrams
[  3] Server Report:
[  3]  0.0-10.0 sec   878 MBytes   737 Mbits/sec   0.000 ms 2146230968/2146857308 (0%)
[  3] 0.00-10.00 sec  14641 datagrams received out-of-order
------------------------------------------------------------
[  3] local 192.168.1.201 port 5001 connected with 192.168.1.59 port 40892
[ ID] Interval       Transfer     Bandwidth        Jitter   Lost/Total Datagrams
[  3]  0.0-10.0 sec   878 MBytes   737 Mbits/sec   0.025 ms 2146230968/2146857308 (1e+02%)
[  3] 0.00-10.00 sec  14641 datagrams received out-of-order
```
### T12
```
iperf -c 192.168.1.201 -u -b 1000m -t 10 -i 1 -f m 
------------------------------------------------------------
Client connecting to 192.168.1.201, UDP port 5001
Sending 1470 byte datagrams, IPG target: 11.76 us (kalman adjust)
UDP buffer size: 0.20 MByte (default)
------------------------------------------------------------
[  3] local 192.168.1.59 port 56113 connected with 192.168.1.201 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0- 1.0 sec   114 MBytes   955 Mbits/sec
[  3]  1.0- 2.0 sec  89.1 MBytes   748 Mbits/sec
[  3]  2.0- 3.0 sec  87.3 MBytes   732 Mbits/sec
[  3]  3.0- 4.0 sec  87.1 MBytes   731 Mbits/sec
[  3]  4.0- 5.0 sec  87.0 MBytes   730 Mbits/sec
[  3]  5.0- 6.0 sec  87.2 MBytes   731 Mbits/sec
[  3]  6.0- 7.0 sec  86.8 MBytes   728 Mbits/sec
[  3]  7.0- 8.0 sec  87.0 MBytes   729 Mbits/sec
[  3]  8.0- 9.0 sec  86.8 MBytes   728 Mbits/sec
[  3]  0.0-10.0 sec   899 MBytes   754 Mbits/sec
[  3] Sent 641552 datagrams
[  3] Server Report:
[  3]  0.0-10.0 sec   899 MBytes   755 Mbits/sec   0.000 ms 2146200544/2146842096 (0%)
[  3] 0.00-10.00 sec  4519 datagrams received out-of-order
------------------------------------------------------------
[  3] local 192.168.1.201 port 5001 connected with 192.168.1.59 port 56113
[ ID] Interval       Transfer     Bandwidth        Jitter   Lost/Total Datagrams
[  3]  0.0-10.0 sec   899 MBytes   755 Mbits/sec   0.025 ms 2146200544/2146842096 (1e+02%)
[  3] 0.00-10.00 sec  4519 datagrams received out-of-order
```
### T16
```
iperf -c 192.168.1.201 -u -b 1000m -t 10 -i 1 -f m 
------------------------------------------------------------
Client connecting to 192.168.1.201, UDP port 5001
Sending 1470 byte datagrams, IPG target: 11.76 us (kalman adjust)
UDP buffer size: 0.20 MByte (default)
------------------------------------------------------------
[  3] local 192.168.1.59 port 47073 connected with 192.168.1.201 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0- 1.0 sec  85.1 MBytes   714 Mbits/sec
[  3]  1.0- 2.0 sec  87.2 MBytes   732 Mbits/sec
[  3]  2.0- 3.0 sec  87.1 MBytes   731 Mbits/sec
[  3]  3.0- 4.0 sec  86.9 MBytes   729 Mbits/sec
[  3]  4.0- 5.0 sec  87.2 MBytes   731 Mbits/sec
[  3]  5.0- 6.0 sec  87.5 MBytes   734 Mbits/sec
[  3]  6.0- 7.0 sec  86.6 MBytes   727 Mbits/sec
[  3]  7.0- 8.0 sec  84.9 MBytes   712 Mbits/sec
[  3]  8.0- 9.0 sec  86.9 MBytes   729 Mbits/sec
[  3]  0.0-10.0 sec   867 MBytes   727 Mbits/sec
[  3] Sent 618181 datagrams
[  3] Server Report:
[  3]  0.0-10.0 sec   867 MBytes   727 Mbits/sec   0.000 ms 2146247286/2146865467 (0%)
[  3] 0.00-10.00 sec  37 datagrams received out-of-order
------------------------------------------------------------
[  3] local 192.168.1.201 port 5001 connected with 192.168.1.59 port 47073
[ ID] Interval       Transfer     Bandwidth        Jitter   Lost/Total Datagrams
[  3]  0.0-10.0 sec   867 MBytes   727 Mbits/sec   0.025 ms 2146247286/2146865467 (1e+02%)
[  3] 0.00-10.00 sec  37 datagrams received out-of-order
```
### T20
```
iperf -c 192.168.1.201 -u -b 1000m -t 10 -i 1 -f m 
------------------------------------------------------------
Client connecting to 192.168.1.201, UDP port 5001
Sending 1470 byte datagrams, IPG target: 11.76 us (kalman adjust)
UDP buffer size: 0.20 MByte (default)
------------------------------------------------------------
[  3] local 192.168.1.59 port 48128 connected with 192.168.1.201 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0- 1.0 sec  87.3 MBytes   732 Mbits/sec
[  3]  1.0- 2.0 sec  87.4 MBytes   733 Mbits/sec
[  3]  2.0- 3.0 sec  86.9 MBytes   729 Mbits/sec
[  3]  3.0- 4.0 sec  87.3 MBytes   732 Mbits/sec
[  3]  4.0- 5.0 sec  87.1 MBytes   731 Mbits/sec
[  3]  5.0- 6.0 sec  87.4 MBytes   733 Mbits/sec
[  3]  6.0- 7.0 sec  84.3 MBytes   707 Mbits/sec
[  3]  7.0- 8.0 sec  86.7 MBytes   727 Mbits/sec
[  3]  8.0- 9.0 sec  86.9 MBytes   729 Mbits/sec
[  3]  0.0-10.0 sec   868 MBytes   728 Mbits/sec
[  3] Sent 619214 datagrams
[  3] Server Report:
[  3]  0.0-10.0 sec   868 MBytes   728 Mbits/sec   0.000 ms 2146245220/2146864434 (0%)
[  3] 0.00-10.00 sec  1734 datagrams received out-of-order
------------------------------------------------------------
[  3] local 192.168.1.201 port 5001 connected with 192.168.1.59 port 48128
[ ID] Interval       Transfer     Bandwidth        Jitter   Lost/Total Datagrams
[  3]  0.0-10.0 sec   868 MBytes   728 Mbits/sec   0.024 ms 2146245220/2146864434 (1e+02%)
[  3] 0.00-10.00 sec  1734 datagrams received out-of-order
```
