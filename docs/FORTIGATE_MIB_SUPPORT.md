# Fortigate SNMP MIB Support Documentation

## Overview

SMON application now supports Fortigate/Fortinet devices with automatic vendor detection based on SNMP `sysDescr` (System Description). The system automatically identifies Fortigate devices and uses appropriate SNMP OIDs from the **FORTINET-FORTIGATE-MIB**.

## Automatic Vendor Detection

The system automatically detects Fortigate devices by checking the System Description (sysDescr) for keywords:
- `fortigate`
- `fortinet`

Once detected, the application automatically uses Fortigate-specific OIDs for monitoring.

## Supported Fortigate MIB Objects

### Standard MIB-II Metrics (Works on all Fortigate models)

| Metric | OID | Description |
|--------|-----|-------------|
| System Description | `1.3.6.1.2.1.1.1.0` | System description string |
| Interface Description | `1.3.6.1.2.1.2.2.1.2.[index]` | Interface name/description |
| Interface RX Octets (32-bit) | `1.3.6.1.2.1.2.2.1.10.[index]` | Received bytes (32-bit counter) |
| Interface TX Octets (32-bit) | `1.3.6.1.2.1.2.2.1.16.[index]` | Transmitted bytes (32-bit counter) |
| Interface RX Octets (64-bit) | `1.3.6.1.2.1.31.1.1.1.6.[index]` | Received bytes (64-bit counter, preferred) |
| Interface TX Octets (64-bit) | `1.3.6.1.2.1.31.1.1.1.10.[index]` | Transmitted bytes (64-bit counter, preferred) |

### Fortigate-Specific Metrics (FORTINET-FORTIGATE-MIB)

| Metric | OID | Description | Data Type | Range |
|--------|-----|-------------|-----------|-------|
| CPU Usage | `1.3.6.1.4.1.12356.101.4.1.3.0` | System CPU usage percentage | Integer | 0-100 |
| Memory Usage | `1.3.6.1.4.1.12356.101.4.1.4.0` | System memory usage percentage | Integer | 0-100 |
| System Uptime | `1.3.6.1.4.1.12356.101.4.1.1.0` | System uptime in hundredths of a second | TimeTicks | - |
| Serial Number | `1.3.6.1.4.1.12356.101.4.1.2.0` | Device serial number | String | - |
| HA Status | `1.3.6.1.4.1.12356.101.13.2.1.0` | High Availability status | Integer | - |
| HA Master Serial | `1.3.6.1.4.1.12356.101.13.2.1.2.0` | Master device serial number (HA mode) | String | - |
| HA Slave Serial | `1.3.6.1.4.1.12356.101.13.2.1.3.0` | Slave device serial number (HA mode) | String | - |

## Data Collection

### Bandwidth Monitoring

The application collects the following bandwidth metrics for each interface:

- **RX Traffic (Inbound)**: Octets received per polling interval
- **TX Traffic (Outbound)**: Octets transmitted per polling interval
- **Automatic Counter Rollover Detection**: Handles 32-bit and 64-bit counter wraps
- **High-Speed Interface Support**: Multi-wrap detection for interfaces with >500 Mbps traffic

### CPU Monitoring

- **CPU Usage**: Real-time system CPU percentage
- **Automatic Detection**: Falls back to standard UCD-SNMP-MIB if Fortigate-specific OID is unavailable
- **Failure Handling**: 
  - Disables CPU polling after 5 consecutive failures
  - Retries CPU polling every 1 hour (device restart detection)

### SNMP Configuration Required

To enable SNMP monitoring on your Fortigate device, configure:

```
config system snmp sysinfo
    set status enable
    set description "FortiGate-<model>"
end

config system snmp community
    edit 1
        set name "public"      # SNMP community string
    next
end
```

For Fortigate OS 7.0+:
```
config system snmp user
    edit "<username>"
        set auth-proto sha
        set auth-pwd <password>
        set priv-proto aes
        set priv-pwd <password>
    next
end
```

## Implementation Details

### Vendor Detection Process

1. System queries `sysDescr` OID (`1.3.6.1.2.1.1.1.0`)
2. Checks if response contains "fortigate" or "fortinet" (case-insensitive)
3. Sets device vendor to `fortigate` if matched
4. Uses Fortigate-specific OIDs for all subsequent queries

### OID Fallback Strategy

For metrics not available via Fortigate-specific OIDs:

1. **Primary**: Try vendor-specific OID
2. **Secondary**: Fall back to standard MIB-II
3. **Tertiary**: Use UCD-SNMP-MIB for CPU (if Fortigate CPU OID unavailable)

### Interface Detection

Fortigate interface naming typically follows these patterns:

| Interface Type | Example | Description |
|----------------|---------|-------------|
| Physical Port | `port1`, `port2`, `port3` | Physical network interfaces |
| Internal Switch | `internal` | Internal switching fabric |
| Virtual Interfaces | `vlan100`, `vlan200` | VLAN interfaces |
| Tunnel Interfaces | `ssl.root`, `ipsec.1` | VPN tunnel endpoints |
| Loopback | `lo` | Loopback interface |
| PPPOE | `pppoe-wan` | PPPoE connections |

## Monitoring Dashboard

The bandwidth dashboard displays:

- **Interface Selection**: Multi-select interfaces per Fortigate device
- **RX/TX Charts**: Combined RX/TX traffic on single chart
- **Statistics Display**:
  - Peak RX/TX bandwidth
  - Average RX/TX bandwidth
  - Current RX/TX bandwidth
- **Time Range Selector**: 1h, 6h, 24h, 7d, 30d
- **Auto-Refresh**: 0s, 30s, 60s, 5min, 10min intervals
- **PDF Report**: Generate and download bandwidth reports

## Known Limitations

1. **Port-based VLANs**: Interface indices may change on configuration updates
2. **HA Mode**: Monitor status of both master and slave devices separately
3. **Software Versions**: MIB objects may vary between FortiOS versions (tested on 6.4+)
4. **SNMP Rate Limiting**: Some Fortigate models limit SNMP query rate - adjust polling interval if needed

## Troubleshooting

### Device Not Detected

```bash
# Test SNMP connectivity
snmpwalk -v2c -c public <fortigate-ip> 1.3.6.1.2.1.1.1.0
```

Expected output should contain "FortiGate" or "Fortinet" string.

### CPU Metric Not Appearing

1. Check logs: `[DeviceID] CPU SNMP not supported - disabling CPU polling`
2. Verify CPU OID is accessible:
```bash
snmpget -v2c -c public <fortigate-ip> 1.3.6.1.4.1.12356.101.4.1.3.0
```
3. Check FortiOS SNMP configuration

### Memory Metric Not Appearing

Memory metric is available via OID `1.3.6.1.4.1.12356.101.4.1.4.0` but requires additional frontend implementation if needed.

## Performance Considerations

- **Polling Interval**: Recommended 60-300 seconds for Fortigate devices
- **SNMP Timeout**: Set to 5-10 seconds for reliable detection
- **Concurrent Polls**: Limit to 5-10 concurrent device polls
- **Interface Limit**: Performance optimal with <100 interfaces per device

## References

- [Fortinet MIB Browser](https://mibbrowser.online/mibdb_search.php?mib=FORTINET-FORTIGATE-MIB)
- [FortiGate Administration Guide](https://docs.fortinet.com/product/fortigate)
- [SNMP Configuration Guide](https://docs.fortinet.com/product/fortigate/latest/snmp-mib-guide)

## Version History

- **v2.0.6** (Current): Added Fortigate MIB support with automatic vendor detection
  - CPU monitoring via `fgSysCpuUsage` OID
  - Memory monitoring via `fgSysMemUsage` OID
  - HA status monitoring support
  - Automatic fallback to standard MIB-II for compatibility
