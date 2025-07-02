# Redis Infrastructure Documentation

## Overview

This repository contains Docker-based Redis infrastructure configurations providing two distinct deployment patterns:

1. **Redis Sentinel Setup** - High-availability Redis cluster with automatic failover
2. **Redis Master-Slave Setup** - Simple replication configuration for basic redundancy

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Redis Sentinel Setup](#redis-sentinel-setup)
- [Redis Master-Slave Setup](#redis-master-slave-setup)
- [Configuration Reference](#configuration-reference)
- [Usage Instructions](#usage-instructions)
- [Connection Details](#connection-details)
- [Monitoring and Maintenance](#monitoring-and-maintenance)
- [Troubleshooting](#troubleshooting)

## Architecture Overview

### Redis Sentinel Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Sentinel 1    │    │   Sentinel 2    │    │   Sentinel 3    │
│   Port: 26379   │    │   Port: 26380   │    │   Port: 26381   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Redis Master  │    │   Redis Slave1  │    │   Redis Slave2  │
│   Port: 6380    │────│   Port: 6381    │    │   Port: 6382    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Redis Master-Slave Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Redis Master  │────│   Redis Slave1  │    │   Redis Slave2  │
│   Port: 6380    │    │   Port: 6381    │    │   Port: 6382    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Redis Sentinel Setup

### Components

The Sentinel setup includes:

- **1 Redis Master** - Primary Redis instance for write operations
- **2 Redis Slaves** - Replica instances for read operations and failover
- **3 Redis Sentinels** - Monitoring and automatic failover management

### Features

- **Automatic Failover**: Sentinels automatically promote a slave to master if the master fails
- **Configuration Provider**: Clients can discover the current master through Sentinels
- **Monitoring**: Continuous health monitoring of all Redis instances
- **Notification**: Event notifications for configuration changes

### Service Details

#### Redis Master
- **Container Name**: `redis-master`
- **Port**: `6380` (host) → `6379` (container)
- **Authentication**: Password required (`pass`)
- **Role**: Primary write instance

#### Redis Slaves
- **Slave 1**:
  - **Container Name**: `redis-slave-1`
  - **Port**: `6381` (host) → `6379` (container)
  - **Replication**: Syncs from `redis-master:6379`
  
- **Slave 2**:
  - **Container Name**: `redis-slave-2`
  - **Port**: `6382` (host) → `6379` (container)
  - **Replication**: Syncs from `redis-master:6379`

#### Redis Sentinels
- **Sentinel 1**:
  - **Container Name**: `redis-sentinel-1`
  - **Port**: `26379` (host) → `26379` (container)
  
- **Sentinel 2**:
  - **Container Name**: `redis-sentinel-2`
  - **Port**: `26380` (host) → `26379` (container)
  
- **Sentinel 3**:
  - **Container Name**: `redis-sentinel-3`
  - **Port**: `26381` (host) → `26379` (container)

## Redis Master-Slave Setup

### Components

The Master-Slave setup includes:

- **1 Redis Master** - Primary Redis instance
- **2 Redis Slaves** - Replica instances

### Features

- **Data Replication**: Automatic data synchronization from master to slaves
- **Read Scaling**: Distribute read operations across slave instances
- **Basic Redundancy**: Manual failover capabilities

### Service Details

#### Redis Master
- **Container Name**: `redis-master`
- **Port**: `6380` (host) → `6379` (container)
- **Authentication**: Password required (`pass`)
- **Role**: Primary read/write instance

#### Redis Slaves
- **Slave 1**:
  - **Container Name**: `redis-slave-1`
  - **Port**: `6381` (host) → `6379` (container)
  - **Role**: Read-only replica
  
- **Slave 2**:
  - **Container Name**: `redis-slave-2`
  - **Port**: `6382` (host) → `6379` (container)
  - **Role**: Read-only replica

## Configuration Reference

### Sentinel Configuration Parameters

The `sentinel.conf` file contains the following key configurations:

| Parameter | Value | Description |
|-----------|-------|-------------|
| `port` | `26379` | Port for Sentinel service |
| `dir` | `/tmp` | Working directory for Sentinel |
| `sentinel resolve-hostnames` | `yes` | Enable hostname resolution (Redis 6.2+) |
| `sentinel monitor mymaster` | `redis-master 6379 2` | Monitor master with quorum of 2 |
| `sentinel auth-pass mymaster` | `pass` | Authentication password for master |
| `sentinel down-after-milliseconds` | `5000` | Time before considering master down |
| `sentinel parallel-syncs` | `1` | Number of slaves to sync simultaneously |
| `sentinel failover-timeout` | `60000` | Maximum time for failover completion |
| `sentinel deny-scripts-reconfig` | `yes` | Disable runtime script reconfiguration |

### Redis Authentication

All Redis instances use the following authentication configuration:

- **Password**: `pass`
- **Master Auth**: `pass` (for slave authentication to master)
- **Require Pass**: `pass` (for client authentication)

## Usage Instructions

### Starting the Sentinel Setup

```bash
# Navigate to sentinel directory
cd sentinel/

# Start all services
docker-compose up -d

# Check service status
docker-compose ps

# View logs
docker-compose logs -f
```

### Starting the Master-Slave Setup

```bash
# Navigate to master-slave directory
cd master-slave/

# Start all services
docker-compose up -d

# Check service status
docker-compose ps

# View logs
docker-compose logs -f
```

### Stopping Services

```bash
# Stop and remove containers
docker-compose down

# Stop and remove containers with volumes
docker-compose down -v
```

## Connection Details

### Connecting to Redis Instances

#### Sentinel Setup

**Via Sentinel (Recommended for Production):**
```bash
# Connect to Sentinel to discover master
redis-cli -h localhost -p 26379

# In Sentinel CLI, discover master
> SENTINEL get-master-addr-by-name mymaster

# Connect to discovered master
redis-cli -h <master-ip> -p <master-port> -a pass
```

**Direct Connection:**
```bash
# Connect to master
redis-cli -h localhost -p 6380 -a pass

# Connect to slave 1
redis-cli -h localhost -p 6381 -a pass

# Connect to slave 2
redis-cli -h localhost -p 6382 -a pass
```

#### Master-Slave Setup

```bash
# Connect to master
redis-cli -h localhost -p 6380 -a pass

# Connect to slave 1 (read-only)
redis-cli -h localhost -p 6381 -a pass

# Connect to slave 2 (read-only)
redis-cli -h localhost -p 6382 -a pass
```

### Application Connection Examples

#### Python (using redis-py with Sentinel)

```python
import redis.sentinel

# Sentinel connection
sentinel = redis.sentinel.Sentinel([
    ('localhost', 26379),
    ('localhost', 26380),
    ('localhost', 26381)
])

# Get master connection
master = sentinel.master_for('mymaster', socket_timeout=0.1, password='pass')

# Get slave connection for reads
slave = sentinel.slave_for('mymaster', socket_timeout=0.1, password='pass')

# Write to master
master.set('key', 'value')

# Read from slave
value = slave.get('key')
```

#### Node.js (using ioredis)

```javascript
const Redis = require('ioredis');

// Sentinel configuration
const sentinel = new Redis({
  sentinels: [
    { host: 'localhost', port: 26379 },
    { host: 'localhost', port: 26380 },
    { host: 'localhost', port: 26381 }
  ],
  name: 'mymaster',
  password: 'pass'
});

// Usage
await sentinel.set('key', 'value');
const value = await sentinel.get('key');
```

## Monitoring and Maintenance

### Health Checks

#### Check Sentinel Status

```bash
# Connect to any Sentinel
redis-cli -h localhost -p 26379

# Check master info
> SENTINEL masters

# Check slave info
> SENTINEL slaves mymaster

# Check Sentinel info
> SENTINEL sentinels mymaster
```

#### Check Replication Status

```bash
# On master
redis-cli -h localhost -p 6380 -a pass INFO replication

# On slave
redis-cli -h localhost -p 6381 -a pass INFO replication
```

### Common Monitoring Commands

| Command | Description |
|---------|-------------|
| `INFO replication` | Show replication status and lag |
| `INFO memory` | Memory usage statistics |
| `INFO stats` | General statistics |
| `CLIENT LIST` | List connected clients |
| `MONITOR` | Real-time command monitoring |
| `SLOWLOG GET` | Retrieve slow query log |

### Log Management

```bash
# View service logs
docker-compose logs <service-name>

# Follow logs in real-time
docker-compose logs -f <service-name>

# View logs with timestamps
docker-compose logs -t <service-name>
```

## Troubleshooting

### Common Issues

#### 1. Sentinel Cannot Resolve Hostnames

**Problem**: Sentinels cannot connect to Redis master using hostname.

**Solution**: 
- Ensure `sentinel resolve-hostnames yes` is set in `sentinel.conf`
- For Redis versions < 6.2, use IP addresses instead of hostnames
- Verify Docker network connectivity

#### 2. Authentication Failures

**Problem**: Redis returns `(error) NOAUTH Authentication required`

**Solution**:
```bash
# Always use -a flag with password
redis-cli -h localhost -p 6380 -a pass
```

#### 3. Slave Synchronization Issues

**Problem**: Slaves are not syncing with master.

**Diagnosis**:
```bash
# Check master replication info
redis-cli -h localhost -p 6380 -a pass INFO replication

# Check slave replication info
redis-cli -h localhost -p 6381 -a pass INFO replication
```

**Solution**:
- Verify network connectivity between containers
- Check authentication credentials
- Restart slave containers if needed

#### 4. Sentinel Failover Not Working

**Problem**: Sentinel doesn't promote slave during master failure.

**Diagnosis**:
```bash
# Check Sentinel logs
docker-compose logs redis-sentinel-1

# Check quorum configuration
redis-cli -h localhost -p 26379 SENTINEL masters
```

**Solution**:
- Ensure quorum (2) is achievable with available Sentinels
- Verify all Sentinels can communicate with Redis instances
- Check `down-after-milliseconds` configuration

### Performance Optimization

#### Memory Optimization

```bash
# Set memory policy (in Redis CLI)
CONFIG SET maxmemory-policy allkeys-lru

# Set memory limit
CONFIG SET maxmemory 256mb
```

#### Persistence Configuration

```bash
# Disable persistence for performance (if acceptable)
CONFIG SET save ""

# Or configure lighter persistence
CONFIG SET save "900 1 300 10 60 10000"
```

### Backup and Recovery

#### Create Backup

```bash
# Create RDB snapshot
redis-cli -h localhost -p 6380 -a pass BGSAVE

# Copy RDB file from container
docker cp redis-master:/data/dump.rdb ./backup/
```

#### Restore from Backup

```bash
# Stop Redis service
docker-compose stop master

# Copy backup to container
docker cp ./backup/dump.rdb redis-master:/data/

# Restart Redis service
docker-compose start master
```

## Security Considerations

1. **Change Default Password**: Replace `pass` with a strong password
2. **Network Security**: Use Docker networks and firewall rules
3. **TLS Encryption**: Consider enabling TLS for production environments
4. **Access Control**: Implement Redis ACLs for fine-grained permissions
5. **Container Security**: Run containers with non-root users when possible

## Environment-Specific Configurations

### Development Environment

- Use default configurations provided
- Enable verbose logging for troubleshooting
- Consider reducing timeout values for faster testing

### Production Environment

- Change default passwords
- Implement monitoring and alerting
- Configure appropriate memory limits
- Set up persistent storage volumes
- Implement backup strategies
- Use TLS encryption
- Configure proper network policies

## API Reference

### Sentinel Commands

| Command | Description | Example |
|---------|-------------|---------|
| `SENTINEL masters` | List monitored masters | `SENTINEL masters` |
| `SENTINEL slaves <master-name>` | List slaves of a master | `SENTINEL slaves mymaster` |
| `SENTINEL get-master-addr-by-name <master-name>` | Get master address | `SENTINEL get-master-addr-by-name mymaster` |
| `SENTINEL failover <master-name>` | Force failover | `SENTINEL failover mymaster` |
| `SENTINEL reset <pattern>` | Reset Sentinel state | `SENTINEL reset mymaster` |

### Redis Replication Commands

| Command | Description | Example |
|---------|-------------|---------|
| `INFO replication` | Show replication info | `INFO replication` |
| `SLAVEOF <host> <port>` | Make instance a slave | `SLAVEOF redis-master 6379` |
| `SLAVEOF NO ONE` | Promote slave to master | `SLAVEOF NO ONE` |
| `ROLE` | Show instance role | `ROLE` |

This documentation provides complete coverage of the Redis infrastructure configurations, including setup procedures, configuration details, monitoring guidance, and troubleshooting information for both development and production environments.