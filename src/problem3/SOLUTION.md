# NGINX Load Balancer Memory Issue Troubleshooting Guide

## Overview
This document outlines the troubleshooting steps, potential causes, impacts, and recovery procedures for a high memory usage issue (99%) on an Ubuntu 24.04 VM running NGINX as a load balancer.

## System Specifications
- **Operating System**: Ubuntu 24.04
- **Storage**: 64GB
- **Service**: NGINX Load Balancer
- **Issue**: 99% Memory Usage
- **Role**: Traffic Router for Upstream Services

## Initial Assessment Steps

### 1. Memory Usage Analysis
```bash
# Some cli command we can use to check current memory usage
free -h
vmstat 1
top

# Check swap usage
swapon --show
```

### 2. NGINX Process Inspection
```bash
# Check NGINX processes
ps aux | grep nginx
pmap -x $(pidof nginx)

# Check NGINX status
systemctl status nginx
nginx -t
```

### 3. Log Analysis
```bash
# View error logs
tail -f /var/log/nginx/error.log

# Check system logs
journalctl -u nginx --since "1 hour ago"
```

## Potential Root Causes

### 1. Memory Leak in NGINX Process

#### Symptoms
- Growing NGINX worker processes
- Increasing RSS (Resident Set Size) over time
- Gradual memory consumption without release

#### Investigation
```bash
# Monitor memory growth
watch -n 1 'ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head'

# Check file descriptors
lsof -p $(pidof nginx)
```

#### Impact
- Degraded performance
- Potential service outage
- Increased response times

#### Recovery Steps
1. Backup configuration
   ```bash
   cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup
   ```
2. Adjust worker processes
   ```bash
   # Edit nginx.conf
   worker_processes auto;
   worker_connections <Max number connection>;
   # Adjust this number base on your machine configuration
   ```
3. Reload NGINX
   ```bash
   nginx -s reload
   ```
4. Update NGINX if needed
   ```bash
   apt update && apt upgrade nginx
   ```

### 2. Configuration Issues

#### Symptoms
- High number of worker processes
- Excessive connection counts
- High memory per worker

#### Investigation
```bash
# Review configuration
nginx -T

# Monitor connections
netstat -antp | grep nginx
ss -s
```

#### Impact
- Resource inefficiency
- Decreased performance
- Unnecessary memory consumption

#### Recovery Steps
1. Optimize worker configuration
   ```nginx
   worker_processes auto;
   worker_rlimit_nofile 65535;
   ```
2. Adjust buffer sizes
   ```nginx
   client_body_buffer_size 10K;
   client_header_buffer_size 1k;
   client_max_body_size 8m;
   large_client_header_buffers 2 1k;
   ```
3. Enable gzip compression
   ```nginx
   gzip on;
   gzip_types text/plain application/json;
   ```

### 3. Connection Pool Exhaustion

#### Symptoms
- Many TIME_WAIT connections
- High number of active connections
- Connection timeouts

#### Investigation
```bash
# Check connection states
netstat -nat | awk '{print $6}' | sort | uniq -c | sort -n

# Monitor connection rate
tcpdump -i any port 80 or port 443 -n
```

#### Impact
- Failed connections
- Increased latency
- Service instability

#### Recovery Steps
1. Adjust system parameters
   ```bash
   # Edit /etc/sysctl.conf
   net.ipv4.tcp_fin_timeout = 30
   net.ipv4.tcp_keepalive_time = 1200
   net.ipv4.tcp_max_tw_buckets = 500000
   ```
2. Configure keepalive
   ```nginx
   keepalive_timeout 65;
   keepalive_requests 100;
   ```

## Long-term Prevention Measures

### 1. Monitoring Setup
- Configure CloudWatch/Prometheus alerts
- Set up memory usage thresholds
- Implement log rotation
- Enable detailed metrics collection

### 2. Regular Maintenance
```bash
# Weekly checks
nginx -t
systemctl status nginx
free -h

# Monthly tasks
apt update && apt upgrade
nginx -T > config_backup_$(date +%Y%m%d).txt
```

### 3. Documentation
- Keep configuration changelog
- Document all optimizations
- Maintain incident reports
- Update runbooks regularly

## Performance Tuning Guidelines

### 1. NGINX Configuration Best Practices
```nginx
# Worker configuration
worker_processes auto;
worker_rlimit_nofile 65535;

# Buffer settings
client_body_buffer_size 10K;
client_header_buffer_size 1k;
client_max_body_size 8m;

# Timeouts
keepalive_timeout 65;
keepalive_requests 100;
```

### 2. System Tuning
```bash
# System limits (/etc/security/limits.conf)
nginx soft nofile 65535
nginx hard nofile 65535

# Kernel parameters (/etc/sysctl.conf)
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.core.somaxconn = 65535
```

## Emergency Response Plan

### 1. Immediate Actions
1. Capture system state
   ```bash
   free -h > memory_state.log
   ps aux > process_state.log
   ```
2. Backup configuration
   ```bash
   cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.$(date +%Y%m%d)
   ```
3. Reload or restart service
   ```bash
   nginx -s reload || systemctl restart nginx
   ```

### 2. Communication Protocol
1. Alert relevant team members
2. Update status page
3. Notify affected users if necessary
4. Document incident timeline
