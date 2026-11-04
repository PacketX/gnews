# GRISM release notes

GRISM firmware comes in two lines, each released and numbered on its own. Pick the one for your model:

| Line | Version | Models | Latest | Release notes |
|---|---|---|---|---|
| DPDK | 7.x | Q16, F3T1G4, T4G12, H2 | 7.6.261028.1 | [DPDK line release notes](/en/release-dpdk.md) |
| MIPS | 6.x | G8S, G8, T12S, T20, F2T12, F4T4 and their MEC variants | 6.6.261029.1 | [MIPS line release notes](/en/release-mips.md) |

The two lines carry the same features, and releases cut on the same day hold the same changes apart from what is specific to each hardware. The version is `<line>.<YYMMDD>.<build of that day>`; `etc/version` and the web page's get_version show what a box runs.
