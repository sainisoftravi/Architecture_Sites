# Physical Machine High Availability Solution for Cupola

## Overview
This document provides practical solutions for physical machine failures in Cupola Docker architecture, focusing on real-world scenarios where machines are physical and not virtualized.

## Physical Machine Failure Scenarios & Solutions

### **Scenario 1: Slave Physical Machine Failure**

#### **Problem**: 
- Physical Slave machine goes down
- All cameras lose RTMP connection
- No automatic failover possible
- Manual intervention required

#### **Solution Options**:

##### **Option A: Hot Standby with Manual Switchover**
```mermaid
graph TB
    subgraph "Physical Machines"
        S1[Slave Machine 1<br/>Primary<br/>IP: 192.168.1.101<br/>Cupola Docker]
        S2[Slave Machine 2<br/>Hot Standby<br/>IP: 192.168.1.102<br/>Cupola Docker]
    end
    
    subgraph "Cameras"
        C1[Camera 1<br/>rtmp://192.168.1.101:1935/stream1]
        C2[Camera 2<br/>rtmp://192.168.1.101:1935/stream2]
    end
    
    subgraph "Network Switch"
        SW[Network Switch<br/>Port Configuration]
    end
    
    C1 -->|RTMP| SW
    C2 -->|RTMP| SW
    SW -->|Active| S1
    SW -.->|Standby| S2
```

**Implementation Steps**:
1. **Keep backup machine running** with Cupola Docker
2. **Sync MongoDB** between machines regularly
3. **When primary fails**:
   - Change network switch port configuration
   - Update camera RTMP URLs (if possible)
   - Start Cupola services on backup
   - **Downtime**: 5-10 minutes

##### **Option B: IP Address Takeover**
```bash
# When Slave 1 (192.168.1.101) fails:
# 1. Change Slave 2 IP to 192.168.1.101
# 2. Restart network services
# 3. Start Cupola Docker
# 4. Cameras automatically reconnect
```

**Steps**:
1. **Physical machine failure** detected
2. **Change backup machine IP** to failed machine's IP
3. **Restart network** services
4. **Start Cupola Docker** containers
5. **Cameras reconnect** automatically (same IP)
6. **Downtime**: 2-3 minutes

##### **Option C: DNS-Based Failover**
```mermaid
graph TB
    subgraph "DNS Server"
        DNS[DNS Server<br/>slave.cupola.local<br/>192.168.1.101]
    end
    
    subgraph "Physical Machines"
        S1[Slave 1<br/>192.168.1.101<br/>Primary]
        S2[Slave 2<br/>192.168.1.102<br/>Backup]
    end
    
    subgraph "Cameras"
        C1[Camera 1<br/>rtmp://slave.cupola.local:1935/stream1]
        C2[Camera 2<br/>rtmp://slave.cupola.local:1935/stream2]
    end
    
    DNS -->|Active| S1
    DNS -.->|Failover| S2
    C1 -->|DNS Lookup| DNS
    C2 -->|DNS Lookup| DNS
```

**Implementation**:
1. **Use DNS names** instead of IP addresses
2. **Configure DNS failover** (PowerDNS, BIND)
3. **When primary fails**: Update DNS record
4. **Cameras reconnect** automatically
5. **Downtime**: 1-2 minutes

### **Scenario 2: Compressor Physical Machine Failure**

#### **Problem**: 
- Compressor machine goes down
- No compression available
- RTSP streams unavailable

#### **Solution**:
```mermaid
graph TB
    subgraph "Slave Machine"
        SF1[Factory 1<br/>RTMP Factory<br/>All Cameras]
        SF2[Factory 2<br/>RTSP Factory<br/>Compressed Streams]
    end
    
    subgraph "Compressor Machines"
        COMP1[Compressor 1<br/>Primary<br/>192.168.1.201]
        COMP2[Compressor 2<br/>Backup<br/>192.168.1.202]
    end
    
    subgraph "Digital Twin Configuration"
        DT1[Mode 1: With Compression<br/>RTSP Output]
        DT2[Mode 2: Without Compression<br/>RTMP Only]
    end
    
    SF1 -->|RTSP URLs| COMP1
    SF1 -.->|Backup| COMP2
    COMP1 -->|Compressed RTSP| SF2
    COMP2 -->|Compressed RTSP| SF2
    
    DT1 -->|Normal| COMP1
    DT2 -->|Fallback| COMP2
```

**Implementation**:
1. **Dual patrol modes** in Digital Twin
2. **Mode 1**: With compression (when compressor available)
3. **Mode 2**: Without compression (when compressor down)
4. **Manual switchover** when compressor fails
5. **No impact** on basic RTMP functionality

### **Scenario 3: Master Physical Machine Failure**

#### **Problem**: 
- Master machine goes down
- No automatic failover
- Manual restoration required

#### **Solution**:
```mermaid
graph TB
    subgraph "Master Machines"
        M1[Master 1<br/>Primary<br/>192.168.1.301<br/>Cupola Docker]
        M2[Master 2<br/>Backup<br/>192.168.1.302<br/>Cupola Docker]
    end
    
    subgraph "Slave Machines"
        S1[Slave 1<br/>RTMP/RTSP Streams]
        S2[Slave 2<br/>RTMP/RTSP Streams]
    end
    
    subgraph "Middleware/Digital Twin"
        MW[Middleware<br/>Port Forwarding<br/>URL Management]
        DT[Digital Twin<br/>Configuration<br/>URL Updates]
    end
    
    S1 -->|Streams| M1
    S2 -->|Streams| M1
    S1 -.->|Backup| M2
    S2 -.->|Backup| M2
    
    M1 -->|Streams| MW
    M2 -.->|Backup| MW
    MW -->|Config| DT
```

**Implementation**:
1. **Keep backup Master running** with Cupola Docker
2. **Sync MongoDB** between Master machines
3. **When primary fails**:
   - Update middleware configuration
   - Change Digital Twin URLs
   - Start services on backup Master
   - **Downtime**: 5-10 minutes

## Complete Physical Machine Architecture

```mermaid
graph TB
    subgraph "📹 Cameras (99+ Cameras)"
        C1[📹 Camera 1<br/>rtmp://192.168.1.101:1935/stream1]
        C2[📹 Camera 2<br/>rtmp://192.168.1.101:1935/stream2]
        CN[📹 Camera N<br/>rtmp://192.168.1.101:1935/streamN]
    end

    subgraph "🖥️ Slave Physical Machines"
        S1[Slave 1<br/>Primary<br/>192.168.1.101<br/>Cupola Docker<br/>MongoDB]
        S2[Slave 2<br/>Hot Standby<br/>192.168.1.102<br/>Cupola Docker<br/>MongoDB]
    end

    subgraph "🖥️ Compressor Physical Machines"
        COMP1[Compressor 1<br/>Primary<br/>192.168.1.201<br/>Docker]
        COMP2[Compressor 2<br/>Backup<br/>192.168.1.202<br/>Docker]
    end

    subgraph "🖥️ Master Physical Machines"
        M1[Master 1<br/>Primary<br/>192.168.1.301<br/>Cupola Docker<br/>MongoDB]
        M2[Master 2<br/>Backup<br/>192.168.1.302<br/>Cupola Docker<br/>MongoDB]
    end

    subgraph "💾 Shared Storage"
        MONGO[MongoDB Replication<br/>Configuration Sync<br/>Backup & Restore]
    end

    subgraph "🔄 Network Infrastructure"
        SW[Network Switch<br/>Port Configuration<br/>VLAN Management]
        DNS[DNS Server<br/>slave.cupola.local<br/>master.cupola.local]
    end

    %% Camera connections
    C1 -->|RTMP| SW
    C2 -->|RTMP| SW
    CN -->|RTMP| SW
    SW -->|Active| S1
    SW -.->|Standby| S2

    %% Slave to compressor
    S1 -->|RTSP| COMP1
    S2 -->|RTSP| COMP2
    COMP1 -->|Compressed RTSP| S1
    COMP2 -->|Compressed RTSP| S2

    %% Slave to master
    S1 -->|Streams| M1
    S2 -->|Streams| M1
    S1 -.->|Backup| M2
    S2 -.->|Backup| M2

    %% MongoDB connections
    S1 <-->|Replication| MONGO
    S2 <-->|Replication| MONGO
    M1 <-->|Replication| MONGO
    M2 <-->|Replication| MONGO

    %% Styling
    classDef camera fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef slave fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef master fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef compressor fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef storage fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef network fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px

    class C1,C2,CN camera
    class S1,S2 slave
    class M1,M2 master
    class COMP1,COMP2 compressor
    class MONGO storage
    class SW,DNS network
```

## Implementation Strategy for Physical Machines

### **Phase 1: Hot Standby Setup**
1. **Deploy backup machines** with same configuration
2. **Keep services running** on backup machines
3. **Set up MongoDB replication** between machines
4. **Configure network switch** for easy failover

### **Phase 2: Monitoring & Alerting**
1. **Deploy monitoring** on each machine
2. **Set up alerts** for machine failures
3. **Create failover scripts** for quick recovery
4. **Test failover procedures** regularly

### **Phase 3: Automation**
1. **Create failover scripts** for each scenario
2. **Set up automated backups** of configurations
3. **Implement health checks** for all services
4. **Document recovery procedures**

## Practical Recovery Procedures

### **Slave Machine Failure Recovery**
```bash
# When Slave 1 (192.168.1.101) fails:
# 1. Detect failure (monitoring alert)
# 2. Change Slave 2 IP to 192.168.1.101
# 3. Restart network services
# 4. Start Cupola Docker containers
# 5. Verify camera connections
# 6. Update monitoring

# Commands:
sudo ip addr del 192.168.1.102/24 dev eth0
sudo ip addr add 192.168.1.101/24 dev eth0
sudo systemctl restart networking
docker-compose up -d
```

### **Master Machine Failure Recovery**
```bash
# When Master 1 (192.168.1.301) fails:
# 1. Detect failure (monitoring alert)
# 2. Change Master 2 IP to 192.168.1.301
# 3. Restart network services
# 4. Start Cupola Docker containers
# 5. Update middleware configuration
# 6. Update Digital Twin URLs

# Commands:
sudo ip addr del 192.168.1.302/24 dev eth0
sudo ip addr add 192.168.1.301/24 dev eth0
sudo systemctl restart networking
docker-compose up -d
# Update middleware and Digital Twin configurations
```

### **Compressor Machine Failure Recovery**
```bash
# When Compressor 1 fails:
# 1. Detect failure (monitoring alert)
# 2. Switch to Mode 2 (without compression)
# 3. Update Digital Twin configuration
# 4. Verify RTMP streams working
# 5. Plan compressor replacement

# Commands:
# Update Digital Twin to Mode 2
# Verify RTMP streams are working
# No immediate action needed for basic functionality
```

## Benefits of Physical Machine Solution

### **✅ Realistic Approach**
- **Works with physical hardware**
- **No virtualization required**
- **Cost-effective** solution

### **✅ Quick Recovery**
- **2-3 minutes** for Slave failover
- **5-10 minutes** for Master failover
- **Immediate** for Compressor (fallback mode)

### **✅ Reliable**
- **Proven approach** for physical machines
- **Easy to implement** and maintain
- **Minimal additional** infrastructure

### **✅ Scalable**
- **Easy to add** more backup machines
- **Flexible configuration** options
- **Room for growth**

This solution is specifically designed for physical machines and provides practical, implementable solutions for your Cupola Docker architecture!
