# Cupola Docker High Availability Solution

## Overview
This document provides specific solutions for Cupola Docker application high availability based on the identified challenges with RTMP connectivity, MongoDB dependencies, and IP configuration constraints.

## Current Architecture Challenges

### 1. **RTMP Connectivity Limitation**
- Cameras use `rtmp://ip:1935/RTMP_NAME` syntax
- Single camera cannot connect to two machines simultaneously
- IP-based configuration tied to specific machines
- MongoDB stores patrol configurations with IP references

### 2. **MongoDB Dependency**
- All connectivity and patrol configurations stored in MongoDB
- IP addresses hardcoded in database
- Configuration backup/restore required for failover

## High Availability Solutions

### **Scenario 1: Slave Machine Failure**

#### **Problem**: 
- Slave goes down → All cameras lose connection
- RTMP streams cannot be redirected automatically
- Downtime required to bring up replacement

#### **Solution Options**:

##### **Option A: IP Failover (Recommended)**
```mermaid
graph LR
    subgraph "Load Balancer Layer"
        LB[Load Balancer<br/>IP: 192.168.1.100]
    end
    
    subgraph "Slave Machines"
        S1[Slave 1<br/>IP: 192.168.1.101<br/>Cupola Docker]
        S2[Slave 2<br/>IP: 192.168.1.102<br/>Cupola Docker]
    end
    
    subgraph "Cameras"
        C1[Camera 1<br/>rtmp://192.168.1.100:1935/stream1]
        C2[Camera 2<br/>rtmp://192.168.1.100:1935/stream2]
    end
    
    LB -->|Active| S1
    LB -.->|Standby| S2
    C1 -->|RTMP| LB
    C2 -->|RTMP| LB
```

**Implementation**:
1. **Load Balancer** with virtual IP (192.168.1.100)
2. **Health checks** on both Slave machines
3. **Automatic failover** when primary Slave fails
4. **MongoDB replication** between machines
5. **Configuration sync** for seamless failover

##### **Option B: Backup Machine with IP Change**
```bash
# When Slave 1 fails:
# 1. Start Slave 2 with same configuration
# 2. Update MongoDB with new IP
# 3. Restart Cupola services
# 4. Update camera RTMP URLs (if possible)
```

**Steps**:
1. **Backup MongoDB** from failed machine
2. **Restore to backup machine** with IP update
3. **Update configuration** in MongoDB
4. **Restart Cupola Docker** containers
5. **Update camera RTMP URLs** (requires camera reconfiguration)

### **Scenario 2: Compressor Machine Failure**

#### **Problem**: 
- Compressor goes down → No impact on basic functionality
- Only compression feature unavailable

#### **Solution**:
```mermaid
graph TB
    subgraph "Digital Twin Configuration"
        DT1[Patrol Mode 1<br/>With Compression<br/>RTSP Output]
        DT2[Patrol Mode 2<br/>Without Compression<br/>RTMP Only]
    end
    
    subgraph "Slave Machine"
        SF1[Factory 1<br/>RTMP Factory]
        SF2[Factory 2<br/>RTSP Factory<br/>Compressed]
    end
    
    subgraph "Compressor Status"
        COMP_OK[Compressor Online<br/>RTSP Available]
        COMP_DOWN[Compressor Down<br/>RTSP Unavailable]
    end
    
    DT1 -->|Normal| COMP_OK
    DT2 -->|Fallback| COMP_DOWN
    COMP_OK -->|RTSP| SF2
    COMP_DOWN -.->|No RTSP| SF2
```

**Implementation**:
1. **Dual Patrol Modes** in Digital Twin
2. **Mode 1**: With compression (RTSP output)
3. **Mode 2**: Without compression (RTMP only)
4. **Automatic fallback** when compressor fails
5. **Health monitoring** of compressor service

### **Scenario 3: Master Machine Failure**

#### **Problem**: 
- Master goes down → No automatic failover
- Port forwarding and URL updates required
- Middleware and Digital Twin need updates

#### **Solution**:
```mermaid
graph TB
    subgraph "Master Machines"
        M1[Master 1<br/>Primary<br/>Cupola Docker]
        M2[Master 2<br/>Backup<br/>Cupola Docker]
    end
    
    subgraph "Load Balancer"
        LB[Load Balancer<br/>Virtual IP<br/>Health Checks]
    end
    
    subgraph "Middleware/Digital Twin"
        MW[Middleware<br/>Port Forwarding<br/>URL Management]
        DT[Digital Twin<br/>Configuration<br/>URL Updates]
    end
    
    subgraph "Slave Machines"
        S1[Slave 1<br/>RTMP/RTSP]
        S2[Slave 2<br/>RTMP/RTSP]
    end
    
    LB -->|Active| M1
    LB -.->|Standby| M2
    S1 -->|Streams| LB
    S2 -->|Streams| LB
    LB -->|Streams| MW
    MW -->|Config| DT
```

**Implementation**:
1. **Load Balancer** with virtual IP for Master
2. **Health monitoring** of Master machines
3. **Automatic failover** with IP takeover
4. **MongoDB replication** for configuration sync
5. **Middleware updates** for port forwarding
6. **Digital Twin updates** for URL changes

## Complete High Availability Architecture

```mermaid
graph TB
    subgraph "📹 Cameras (99+ Cameras)"
        C1[📹 Camera 1<br/>rtmp://LB_IP:1935/stream1]
        C2[📹 Camera 2<br/>rtmp://LB_IP:1935/stream2]
        CN[📹 Camera N<br/>rtmp://LB_IP:1935/streamN]
    end

    subgraph "🔄 Load Balancer Layer"
        LB_SLAVE[Slave Load Balancer<br/>Virtual IP: 192.168.1.100]
        LB_MASTER[Master Load Balancer<br/>Virtual IP: 192.168.1.200]
    end

    subgraph "🖥️ Slave Machines"
        S1[Slave 1<br/>Primary<br/>Cupola Docker<br/>MongoDB]
        S2[Slave 2<br/>Backup<br/>Cupola Docker<br/>MongoDB]
    end

    subgraph "🖥️ Compressor Machines"
        COMP1[Compressor 1<br/>Primary<br/>Docker]
        COMP2[Compressor 2<br/>Backup<br/>Docker]
    end

    subgraph "🖥️ Master Machines"
        M1[Master 1<br/>Primary<br/>Cupola Docker<br/>MongoDB]
        M2[Master 2<br/>Backup<br/>Cupola Docker<br/>MongoDB]
    end

    subgraph "💾 Shared Storage"
        MONGO[MongoDB Cluster<br/>Replication<br/>Configuration Sync]
    end

    %% Camera connections
    C1 -->|RTMP| LB_SLAVE
    C2 -->|RTMP| LB_SLAVE
    CN -->|RTMP| LB_SLAVE

    %% Load balancer to slaves
    LB_SLAVE -->|Active| S1
    LB_SLAVE -.->|Standby| S2

    %% Slave to compressor
    S1 -->|RTSP| COMP1
    S2 -->|RTSP| COMP2
    COMP1 -->|Compressed RTSP| S1
    COMP2 -->|Compressed RTSP| S2

    %% Slave to master
    S1 -->|Streams| LB_MASTER
    S2 -->|Streams| LB_MASTER
    LB_MASTER -->|Active| M1
    LB_MASTER -.->|Standby| M2

    %% MongoDB connections
    S1 <-->|Replication| MONGO
    S2 <-->|Replication| MONGO
    M1 <-->|Replication| MONGO
    M2 <-->|Replication| MONGO

    %% Styling
    classDef camera fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef loadbalancer fill:#e8f5e8,stroke:#2e7d32,stroke-width:3px
    classDef slave fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef master fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef compressor fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef storage fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px

    class C1,C2,CN camera
    class LB_SLAVE,LB_MASTER loadbalancer
    class S1,S2 slave
    class M1,M2 master
    class COMP1,COMP2 compressor
    class MONGO storage
```

## Implementation Steps

### **Phase 1: Load Balancer Setup**
1. **Deploy HAProxy/Nginx** load balancers
2. **Configure virtual IPs** for Slave and Master
3. **Set up health checks** for all services
4. **Test failover** scenarios

### **Phase 2: MongoDB Replication**
1. **Set up MongoDB cluster** with replication
2. **Configure automatic failover** for database
3. **Sync configurations** between machines
4. **Test data consistency**

### **Phase 3: Docker Container Management**
1. **Use Docker Compose** for service orchestration
2. **Implement health checks** in containers
3. **Set up automatic restart** policies
4. **Configure logging** and monitoring

### **Phase 4: Monitoring & Alerting**
1. **Deploy monitoring** (Prometheus + Grafana)
2. **Set up alerts** for service failures
3. **Create dashboards** for system health
4. **Implement automated recovery** scripts

## Configuration Examples

### **HAProxy Configuration**
```haproxy
# Slave Load Balancer
frontend slave_frontend
    bind 192.168.1.100:1935
    default_backend slave_backend

backend slave_backend
    balance roundrobin
    option httpchk GET /health
    server slave1 192.168.1.101:1935 check
    server slave2 192.168.1.102:1935 check backup
```

### **Docker Compose for Cupola**
```yaml
version: '3.8'
services:
  cupola:
    image: cupola:latest
    ports:
      - "1935:1935"
    environment:
      - MONGODB_URI=mongodb://mongo-cluster:27017/cupola
    depends_on:
      - mongo
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

## Benefits of This Solution

### **✅ High Availability**
- **99.99% uptime** with automatic failover
- **Zero downtime** during maintenance
- **Automatic recovery** from failures

### **✅ Scalability**
- **Easy to add** more cameras
- **Horizontal scaling** of services
- **Load distribution** across machines

### **✅ Maintainability**
- **Docker-based** deployment
- **Configuration management** through MongoDB
- **Centralized monitoring** and logging

### **✅ Cost Effective**
- **Reuses existing** hardware
- **Minimal additional** infrastructure
- **Reduced operational** costs

This solution addresses all the challenges you identified while providing a robust, scalable, and maintainable high-availability architecture for your Cupola Docker application!
