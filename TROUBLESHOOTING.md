# TROUBLESHOOTING GUIDE

## Common Issues and Solutions

This guide provides systematic solutions for common problems encountered during OAI 5G deployment.

## Table of Contents

1. [Pod and Container Issues](#pod-and-container-issues)
2. [Network Connectivity Problems](#network-connectivity-problems)
3. [Service Discovery Failures](#service-discovery-failures)
4. [5G Registration Issues](#5g-registration-issues)
5. [Data Plane Connectivity](#data-plane-connectivity)
6. [Resource and Performance Issues](#resource-and-performance-issues)
7. [Advanced Debugging](#advanced-debugging)

---

## Pod and Container Issues

### Problem: Pods Stuck in Init State

**Symptoms:**
```bash
kubectl get pods -n oai-tutorial
NAME              READY   STATUS     RESTARTS   AGE
oai-upf-xxx       0/1     Init:0/1   0          5m
```

**Diagnosis:**
```bash
# Check init container logs
kubectl logs -n oai-tutorial <pod-name> -c init

# Check events
kubectl describe pod <pod-name> -n oai-tutorial | grep Events -A 10
```

**Solutions:**

1. **Remove Problematic Init Container:**
```bash
kubectl patch deployment $(kubectl get deployment -n oai-tutorial | grep upf | awk '{print $1}') \
  -n oai-tutorial --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/initContainers", "value": []}]'
```

2. **Fix NRF Connection Timeout:**
```bash
# Check NRF service
kubectl get svc oai-nrf -n oai-tutorial
kubectl get endpoints oai-nrf -n oai-tutorial

# Restart NRF if needed
kubectl delete pod $(kubectl get pods -n oai-tutorial | grep nrf | awk '{print $1}') -n oai-tutorial
```

### Problem: Container Restart Loops

**Symptoms:**
```bash
kubectl get pods -n oai-tutorial
NAME              READY   STATUS    RESTARTS   AGE
oai-nr-ue-xxx     1/1     Running   6          10m
```

**Diagnosis:**
```bash
# Check restart reason
kubectl describe pod <pod-name> -n oai-tutorial | grep -A 10 -B 10 "Last State"

# Check previous logs
kubectl logs -n oai-tutorial <pod-name> -c <container-name> --previous --tail=20
```

**Solutions:**

1. **Increase Resource Limits:**
```bash
kubectl patch deployment <deployment-name> -n oai-tutorial --type='json' -p='[{
  "op": "replace",
  "path": "/spec/template/spec/containers/0/resources",
  "value": {
    "limits": {"memory": "2Gi", "cpu": "1000m"},
    "requests": {"memory": "512Mi", "cpu": "200m"}
  }
}]'
```

2. **Fix Liveness Probe Issues:**
```bash
# Remove problematic probes temporarily
kubectl patch deployment <deployment-name> -n oai-tutorial --type='json' \
  -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/livenessProbe"}]'
```

### Problem: ImagePullBackOff Errors

**Symptoms:**
```bash
kubectl get pods -n oai-tutorial
NAME              READY   STATUS             RESTARTS   AGE
oai-amf-xxx       0/1     ImagePullBackOff   0          2m
```

**Solutions:**

1. **Create Docker Registry Secret:**
```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n oai-tutorial
```

2. **Update Deployment to Use Secret:**
```bash
kubectl patch deployment <deployment-name> -n oai-tutorial -p='
{
  "spec": {
    "template": {
      "spec": {
        "imagePullSecrets": [{"name": "regcred"}]
      }
    }
  }
}'
```

---

## Network Connectivity Problems

### Problem: Service Discovery Failures

**Symptoms:**
```bash
kubectl logs -n oai-tutorial <pod-name> | grep "connection refused\|timeout"
```

**Diagnosis:**
```bash
# Check DNS resolution
kubectl exec -n oai-tutorial <pod-name> -- nslookup <service-name>

# Check service endpoints
kubectl get endpoints -n oai-tutorial
```

**Solutions:**

1. **Fix CoreDNS Issues:**
```bash
# Check CoreDNS status
kubectl get pods -n kube-system | grep coredns

# Restart CoreDNS
kubectl delete pod -n kube-system -l k8s-app=kube-dns

# Wait for restart
kubectl wait --for=condition=ready pod -l k8s-app=kube-dns -n kube-system --timeout=300s
```

2. **Fix Service Endpoint Mismatches:**
```bash
# Check actual pod IPs
kubectl get pods -n oai-tutorial -o wide

# Check service endpoints
kubectl get endpoints -n oai-tutorial

# Force endpoint update
kubectl delete endpoints <service-name> -n oai-tutorial
# Service will recreate with correct IP
```

3. **Manual Hosts File Update:**
```bash
# Get correct IP
POD_IP=$(kubectl get pod <pod-name> -n oai-tutorial -o jsonpath='{.status.podIP}')

# Update hosts file in client pod
kubectl exec -n oai-tutorial <client-pod> -- sh -c \
  "echo '$POD_IP <service-name>' >> /etc/hosts"
```

### Problem: CNI Network Issues

**Symptoms:**
- Pods can't communicate with each other
- Network timeouts between services

**Diagnosis:**
```bash
# Check Calico pods
kubectl get pods -n kube-system | grep calico

# Check node network configuration
kubectl describe node <node-name> | grep "PodCIDR\|InternalIP"
```

**Solutions:**

1. **Restart Calico Network:**
```bash
# Restart Calico pods
kubectl delete pod -n kube-system -l k8s-app=calico-node

# Wait for restart
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n kube-system --timeout=300s
```

2. **Fix Node IP Annotations:**
```bash
# Check current annotation
kubectl get node <node-name> -o yaml | grep "projectcalico.org/IPv4Address"

# Update if incorrect
kubectl annotate node <node-name> projectcalico.org/IPv4Address=<correct-ip>/24 --overwrite
```

### Problem: Inter-Pod Communication Failures

**Symptoms:**
```bash
kubectl exec -n oai-tutorial <pod-a> -- ping <pod-b-ip>
# 100% packet loss
```

**Solutions:**

1. **Check Network Policies:**
```bash
# List network policies
kubectl get networkpolicies -n oai-tutorial
kubectl get networkpolicies --all-namespaces

# Remove if blocking
kubectl delete networkpolicy <policy-name> -n oai-tutorial
```

2. **Test Basic Connectivity:**
```bash
# Test DNS resolution
kubectl exec -n oai-tutorial <pod-name> -- nslookup google.com

# Test external connectivity
kubectl exec -n oai-tutorial <pod-name> -- ping -c 2 8.8.8.8

# Test internal connectivity
kubectl exec -n oai-tutorial <pod-name> -- ping -c 2 <other-pod-ip>
```

---

## Service Discovery Failures

### Problem: gNB Cannot Connect to AMF

**Symptoms:**
```bash
kubectl logs -n oai-tutorial <gnb-pod> | grep "SCTP\|AMF\|connect"
# Shows connection failures
```

**Diagnosis:**
```bash
# Check AMF service
kubectl get svc oai-amf -n oai-tutorial
kubectl get endpoints oai-amf -n oai-tutorial

# Test connectivity from gNB
kubectl exec -n oai-tutorial <gnb-pod> -- ping <amf-ip>
```

**Solutions:**

1. **Update gNB Configuration:**
```bash
# Check current AMF IP in gNB config
kubectl get configmap oai-gnb-configmap -n oai-tutorial -o yaml | grep amf_ip_address

# Get correct AMF IP
AMF_IP=$(kubectl get pod <amf-pod> -n oai-tutorial -o jsonpath='{.status.podIP}')

# Update service endpoint if needed
kubectl patch endpoints oai-amf -n oai-tutorial --type='json' \
  -p="[{\"op\": \"replace\", \"path\": \"/subsets/0/addresses/0/ip\", \"value\": \"$AMF_IP\"}]"
```

2. **Restart gNB:**
```bash
kubectl delete pod $(kubectl get pods -n oai-tutorial | grep gnb | awk '{print $1}') -n oai-tutorial
```

### Problem: UE Cannot Connect to gNB RF Simulator

**Symptoms:**
```bash
kubectl logs -n oai-tutorial <ue-pod> -c nr-ue | grep "connect() to oai-ran:4043 failed"
```

**Solutions:**

1. **Update UE Hosts File:**
```bash
# Get current gNB IP
GNB_IP=$(kubectl get pod <gnb-pod> -n oai-tutorial -o jsonpath='{.status.podIP}')

# Update UE hosts file
kubectl exec -n oai-tutorial <ue-pod> -c nr-ue -- sh -c \
  "grep -v 'oai-ran' /etc/hosts > /tmp/hosts_new && echo '$GNB_IP oai-ran' >> /tmp/hosts_new && cp /tmp/hosts_new /etc/hosts"

# Verify update
kubectl exec -n oai-tutorial <ue-pod> -c nr-ue -- cat /etc/hosts | grep oai-ran
```

2. **Test RF Port Connectivity:**
```bash
# Test port access
kubectl exec -n oai-tutorial <ue-pod> -c nr-ue -- timeout 5 bash -c \
  "</dev/tcp/oai-ran/4043" 2>/dev/null && echo "Port open" || echo "Port closed"
```

---

## 5G Registration Issues

### Problem: UE Not Registering with AMF

**Symptoms:**
```bash
kubectl logs -n oai-tutorial <amf-pod> --tail=50 | grep -A 10 "UEs' Information"
# Shows empty UE table
```

**Diagnosis:**
```bash
# Check gNB connection status
kubectl logs -n oai-tutorial <amf-pod> --tail=50 | grep -A 10 "gNBs' Information"

# Check UE logs for registration attempts
kubectl logs -n oai-tutorial <ue-pod> -c nr-ue --tail=100 | grep -i "registration\|attach"
```

**Solutions:**

1. **Verify gNB-AMF Connection:**
```bash
# Check if gNB shows as "Connected" in AMF
kubectl logs -n oai-tutorial <amf-pod> --tail=50 | grep -A 10 "gNBs' Information"

# If "Disconnected", restart gNB
kubectl delete pod $(kubectl get pods -n oai-tutorial | grep gnb | awk '{print $1}') -n oai-tutorial
```

2. **Check Subscriber Database:**
```bash
# Verify subscriber exists in database
kubectl exec -n oai-tutorial <mysql-pod> -- mysql -u root -p<password> -e \
  "SELECT * FROM oai_db.AuthenticationSubscription WHERE ueid='001010000000100';"
```

3. **Verify UE Configuration:**
```bash
# Check UE IMSI matches database
kubectl logs -n oai-tutorial <ue-pod> -c nr-ue | grep "IMSI\|UICC simulation"
```

### Problem: RRC Connection Failures

**Symptoms:**
```bash
kubectl logs -n oai-tutorial <ue-pod> -c nr-ue | grep "RRC"
# Shows RRC setup failures
```

**Solutions:**

1. **Check RF Simulator Connection:**
```bash
# Verify RF connection is established
kubectl logs -n oai-tutorial <ue-pod> -c nr-ue | grep "Connection.*established"

# Check gNB frame processing
kubectl logs -n oai-tutorial <gnb-pod> --tail=20 | grep "Frame.Slot"
```

2. **Restart UE:**
```bash
kubectl delete pod $(kubectl get pods -n oai-tutorial | grep nr-ue | awk '{print $1}') -n oai-tutorial
```

---

## Data Plane Connectivity

### Problem: PDU Session Not Established

**Symptoms:**
- UE registered but no internet connectivity
- Tunnel interface missing or down

**Diagnosis:**
```bash
# Check PDU session logs in SMF
kubectl logs -n oai-tutorial <smf-pod> --tail=50 | grep -i "pdu\|session"

# Check tunnel interface
kubectl exec -n oai-tutorial <ue-pod> -c nr-ue -- ip addr show oaitun_ue1
```

**Solutions:**

1. **Force PDU Session Creation:**
```bash
# Generate traffic to trigger session
kubectl exec -n oai-tutorial <ue-pod> -c nr-ue -- ping -c 1 8.8.8.8 &

# Monitor SMF for session creation
kubectl logs -n oai-tutorial <smf-pod> -f | grep -i "pdu\|session"
```

2. **Check SMF-UPF Communication:**
```bash
# Verify PFCP session
kubectl logs -n oai-tutorial <smf-pod> | grep 'handle_receive(16 bytes)' | wc -l
kubectl logs -n oai-tutorial <upf-pod> | grep 'handle_receive(16 bytes)' | wc -l
# Both should be > 1
```

### Problem: Tunnel Interface Has No Internet Access

**Symptoms:**
```bash
kubectl exec -n oai-tutorial <ue-pod> -c nr-ue -- ping -c 3 -I oaitun_ue1 8.8.8.8
# 100% packet loss
```

**Solutions:**

1. **Configure UPF Routing:**
```bash
# Enable IP forwarding
kubectl exec -n oai-tutorial <upf-pod> -- sysctl -w net.ipv4.ip_forward=1

# Add NAT rules
kubectl exec -n oai-tutorial <upf-pod> -- iptables -t nat -A POSTROUTING -s 12.1.1.0/24 -o eth0 -j MASQUERADE

# Add forwarding rules
kubectl exec -n oai-tutorial <upf-pod> -- iptables -A FORWARD -i tun0 -o eth0 -j ACCEPT
```

2. **Configure UE Tunnel:**
```bash
# Bring tunnel interface up
kubectl exec -n oai-tutorial <ue-pod> -c nr-ue -- ip link set oaitun_ue1 up

# Add IP if missing
kubectl exec -n oai-tutorial <ue-pod> -c nr-ue -- ip addr add 12.1.1.100/24 dev oaitun_ue1

# Add routing
kubectl exec -n oai-tutorial <ue-pod> -c nr-ue -- ip route add 8.8.8.8/32 dev oaitun_ue1
```

3. **Debug Packet Flow:**
```bash
# Check if packets reach UPF
kubectl exec -n oai-tutorial <upf-pod> -- tcpdump -i tun0 -c 5 &
kubectl exec -n oai-tutorial <ue-pod> -c nr-ue -- ping -c 2 -I oaitun_ue1 8.8.8.8

# Check UPF interface statistics
kubectl exec -n oai-tutorial <upf-pod> -- netstat -i
```

---

## Resource and Performance Issues

### Problem: High CPU/Memory Usage

**Symptoms:**
```bash
kubectl top pods -n oai-tutorial
# Shows high resource consumption
```

**Solutions:**

1. **Optimize Resource Limits:**
```bash
# Set appropriate limits for gNB
kubectl patch deployment <gnb-deployment> -n oai-tutorial --type='json' -p='[{
  "op": "replace",
  "path": "/spec/template/spec/containers/0/resources",
  "value": {
    "limits": {"memory": "2Gi", "cpu": "1000m"},
    "requests": {"memory": "1Gi", "cpu": "500m"}
  }
}]'
```

2. **Check Resource Availability:**
```bash
# Check node resources
kubectl top nodes
kubectl describe node <node-name> | grep -A 10 "Allocated resources"
```

### Problem: Pod Eviction Due to Resource Pressure

**Symptoms:**
```bash
kubectl get events -n oai-tutorial | grep "Evicted"
```

**Solutions:**

1. **Clean Up Old Deployments:**
```bash
# Remove old namespaces
kubectl get namespaces | grep oai
kubectl delete namespace <old-namespace>

# Clean up unused images
docker system prune -f
```

2. **Increase Node Resources:**
- Add more CPU/memory to Kubernetes nodes
- Scale to multi-node cluster if needed

---

## Advanced Debugging

### Packet Capture and Analysis

```bash
# Capture packets in network functions
kubectl exec -it -c tcpdump <nf-pod> -n oai-tutorial -- /bin/sh
tcpdump -i any -w /tmp/capture.pcap

# Copy capture file
kubectl cp oai-tutorial/<pod-name>:/tmp/capture.pcap ./capture.pcap -c tcpdump
```

### Complete Health Check Script

```bash
#!/bin/bash
echo "=== 5G Network Debug Information ==="

echo "1. Pod Status:"
kubectl get pods -n oai-tutorial -o wide

echo "2. Service Endpoints:"
kubectl get endpoints -n oai-tutorial

echo "3. Resource Usage:"
kubectl top pods -n oai-tutorial 2>/dev/null || echo "Metrics server not available"

echo "4. Recent Events:"
kubectl get events -n oai-tutorial --sort-by='.lastTimestamp' | tail -10

echo "5. UE Registration:"
kubectl logs -n oai-tutorial $(kubectl get pods -n oai-tutorial | grep amf | awk '{print $1}') \
  | grep "5GMM-REGISTERED" | tail -1 || echo "No registered UEs"

echo "6. gNB Connection:"
kubectl logs -n oai-tutorial $(kubectl get pods -n oai-tutorial | grep amf | awk '{print $1}') \
  | grep -A 5 "gNBs' Information" | tail -5

echo "7. PDU Sessions:"
kubectl logs -n oai-tutorial $(kubectl get pods -n oai-tutorial | grep smf | awk '{print $1}') \
  | grep "PDU_SESSION_ACTIVE" | wc -l

echo "=== Debug Complete ==="
```

### Log Collection for Support

```bash
#!/bin/bash
mkdir -p debug-logs/$(date +%Y%m%d_%H%M%S)
cd debug-logs/$(date +%Y%m%d_%H%M%S)

# Collect all pod logs
for pod in $(kubectl get pods -n oai-tutorial -o name); do
    pod_name=$(echo $pod | cut -d/ -f2)
    kubectl logs -n oai-tutorial $pod_name > ${pod_name}.log 2>&1
done

# Collect pod descriptions
kubectl describe pods -n oai-tutorial > pod-descriptions.txt

# Collect service and endpoint info
kubectl get svc,endpoints -n oai-tutorial -o yaml > services-endpoints.yaml

# Collect events
kubectl get events -n oai-tutorial > events.txt

echo "Debug logs collected in: $(pwd)"
```

---

## Understanding Normal vs Abnormal Behavior

### ✅ Normal 5G Behavior (Not Issues)
- **Tunnel Interface Cycling**: oaitun_ue1 appears/disappears due to 5G DRX power management
- **RF Connection Periodic Errors**: errno(101) during power management cycles
- **Container Restarts**: Occasional restarts due to 5G protocol state management
- **High Log Verbosity**: Technical logs are expected to be verbose

### ❌ Actual Issues to Address
- **Persistent 100% Packet Loss**: On tunnel interface indicates routing problems
- **Pods Stuck in Init**: Configuration or dependency issues need resolution
- **Service Discovery Failures**: DNS resolution problems affecting communication
- **Empty UE Registration Table**: No UEs registered indicates fundamental connectivity issues
- **Constant Pod Restart Loops**: Resource or configuration problems requiring fixes

---

## Getting Additional Help

### Log Analysis Checklist
1. Check pod status and restart counts
2. Examine recent events for error patterns  
3. Verify service discovery and DNS resolution
4. Confirm network function registration status
5. Validate 5G protocol state progression
6. Test basic connectivity between components

### When to Seek Community Support
- Provide debug logs and pod descriptions
- Include exact error messages and symptoms
- Specify software versions and deployment method
- Share relevant configuration files (sanitized)

### Useful Commands Reference
```bash
# Quick status check
kubectl get pods,svc,endpoints -n oai-tutorial

# Resource monitoring
watch kubectl top pods -n oai-tutorial

# Live log monitoring
kubectl logs -f -n oai-tutorial <pod-name> -c <container-name>

# Debug shell access
kubectl exec -it -n oai-tutorial <pod-name> -c <container-name> -- /bin/bash
```

---

*This troubleshooting guide is based on real deployment experiences and covers the most common issues encountered in OAI 5G deployments.*
