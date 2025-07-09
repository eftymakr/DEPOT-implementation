# Complete gNBsim Deployment Guide

## Deployment Options

This guide covers **two deployment methods**:
- **Option A: Docker Compose** (Official OAI tutorial approach)
- **Option B: Kubernetes/Helm** (Your preferred approach)

---

## Docker Compose Deployment (Your Alternative Method)

### Prerequisites Setup

```bash
# Set environment variable for OAI CN5G Fed directory
export OAI_CN5G_FED_DIR=~/oai-cn5g-fed

# Create logs directory
mkdir -p /tmp/oai/mini-gnbsim
chmod 777 /tmp/oai/mini-gnbsim

# Verify Docker environment
docker --version
docker-compose --version
```

### A.1: Clone OAI CN5G Repository

```bash
# Clone the official repository
git clone https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed.git
cd oai-cn5g-fed/docker-compose
```

### A.2: Deploy 5G Core Network (Your Method)

#### Option A1: Basic NRF with eBPF
```bash
# Navigate to docker-compose directory
cd $OAI_CN5G_FED_DIR/docker-compose

# Deploy using basic NRF with eBPF
docker-compose -f docker-compose-basic-nrf-ebpf.yaml up -d

# Verify deployment
docker-compose -f docker-compose-basic-nrf-ebpf.yaml ps
```

#### Option A2: Using Core Network Script
```bash
# Deploy basic 5G core using Python script
python3 ./core-network.py --type start-basic

# OR deploy minimal 5G core (without PCAP capture)
python3 ./core-network.py --type start-mini --scenario 2

# OR deploy with PCAP capture for analysis
python3 ./core-network.py --type start-mini --scenario 2 --capture /tmp/oai/mini-gnbsim/mini-gnbsim.pcap

# Verify all components are running
docker ps -a
```

Expected containers:
- `oai-upf` (User Plane Function)
- `oai-smf` (Session Management Function)  
- `oai-amf` (Access and Mobility Management Function)
- `oai-ext-dn` (External Data Network)
- `mysql` (Database)

### A.3: Build/Get gNBsim Image

#### Option A1: Build from source
```bash
# Clone gnbsim repository (outside oai-cn5g-fed)
cd ~
git clone https://gitlab.eurecom.fr/kharade/gnbsim.git
cd gnbsim

# Build Docker image
docker build --tag gnbsim:latest --target gnbsim --file docker/Dockerfile.ubuntu.22.04 .
```

#### Option A2: Use pre-built image
```bash
docker pull rohankharade/gnbsim
docker image tag rohankharade/gnbsim:latest gnbsim:latest
```

### A.4: Deploy gNBsim (Your Method)

```bash
# Navigate to docker-compose directory
cd $OAI_CN5G_FED_DIR/docker-compose

# Launch gNBsim using your command
docker-compose -f docker-compose-gnbsim.yaml up -d gnbsim

# Wait for healthy status
docker-compose -f docker-compose-gnbsim.yaml ps -a

# Check all services
docker ps -a

# Check logs
docker logs gnbsim
```

### A.5: Validate UE Registration

```bash
# Check UE IP address allocation
docker logs gnbsim 2>&1 | grep "UE address:"
# Expected output: UE address: 12.1.1.2

# Check AMF logs for UE registration
docker logs oai-amf | grep -E "(5GMM-REGISTERED|UE.*registered)"
```

### A.6: Connectivity Tests

#### Ping Test (External DN → UE)
```bash
docker exec oai-ext-dn ping -c 3 12.1.1.2
```

#### Ping Test (UE → Internet)
```bash
docker exec gnbsim ping -c 3 -I 12.1.1.2 google.com
```

#### iperf Performance Test
```bash
# Terminal 1: Start iperf server on external DN
docker exec -it oai-ext-dn iperf3 -s

# Terminal 2: Run iperf client from UE
docker exec -it gnbsim iperf3 -c 192.168.70.135 -B 12.1.1.2
```

### A.7: Scale Testing (Multiple UEs)

```bash
# Deploy additional gNBsim instances
docker-compose -f docker-compose-gnbsim.yaml up -d gnbsim2
docker-compose -f docker-compose-gnbsim.yaml up -d gnbsim3
docker-compose -f docker-compose-gnbsim.yaml up -d gnbsim4
docker-compose -f docker-compose-gnbsim.yaml up -d gnbsim5

# Check UE IP addresses
docker logs gnbsim2 2>&1 | grep "UE address:"
docker logs gnbsim3 2>&1 | grep "UE address:"
docker logs gnbsim4 2>&1 | grep "UE address:"
docker logs gnbsim5 2>&1 | grep "UE address:"

# Verify all UEs are registered
docker logs oai-amf | tail -50
```

### A.8: Log Collection

```bash
# Collect all logs
docker logs oai-amf > /tmp/oai/mini-gnbsim/amf.log 2>&1
docker logs oai-smf > /tmp/oai/mini-gnbsim/smf.log 2>&1
docker logs oai-upf > /tmp/oai/mini-gnbsim/upf.log 2>&1
docker logs oai-ext-dn > /tmp/oai/mini-gnbsim/ext-dn.log 2>&1
docker logs gnbsim > /tmp/oai/mini-gnbsim/gnbsim.log 2>&1
docker logs gnbsim2 > /tmp/oai/mini-gnbsim/gnbsim2.log 2>&1
docker logs gnbsim3 > /tmp/oai/mini-gnbsim/gnbsim3.log 2>&1
docker logs gnbsim4 > /tmp/oai/mini-gnbsim/gnbsim4.log 2>&1
docker logs gnbsim5 > /tmp/oai/mini-gnbsim/gnbsim5.log 2>&1
```

### A.9: Cleanup

```bash
# Remove gNBsim containers
docker-compose -f docker-compose-gnbsim.yaml down -t 0

# Remove 5G core network
python3 ./core-network.py --type stop-mini --scenario 2
```

---

## Option B: Kubernetes/Helm Deployment

### Prerequisites Setup

```bash
# Create logs directory
mkdir -p /tmp/oai/mini-gnbsim
chmod 777 /tmp/oai/mini-gnbsim

# Verify Kubernetes cluster
kubectl cluster-info
kubectl get nodes

# Check available resources
kubectl top nodes

# Verify Helm installation
helm version
```

### B.1: Environment Setup

```bash
# Create namespace for gNBsim deployment
kubectl create namespace oai-gnbsim

# Verify namespace creation
kubectl get namespaces | grep gnbsim
```

### B.2: Repository and Chart Setup

#### Option B1: Using Official OAI Helm Repository
```bash
# Add official OAI Helm repository
helm repo add oai https://charts.oai-ran.org/

# Update repository information
helm repo update

# List available OAI charts
helm search repo oai
```

#### Option B2: Using OAI CN5G Fed Repository (Recommended)
```bash
# Clone the official OAI CN5G Fed repository
git clone https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed.git
cd ~/oai-cn5g-fed/charts

# Navigate to the specific gnbsim scenario
cd gnbsim-chart/charts/e2e_scenarios/case1

# Update Helm dependencies
helm dependency update
```

### B.3: Deploy 5G Core Network

#### Deploy MySQL Database
```bash
# Deploy MySQL for subscriber data
helm install mysql oai/mysql -n oai-gnbsim \
  --set mysql.image.tag=8.0 \
  --set mysql.config.rootPassword=linux \
  --set mysql.config.database=oai_db

# Verify MySQL deployment
kubectl get pods -n oai-gnbsim | grep mysql
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep mysql | awk '{print $1}')
```

#### Deploy NRF (Network Repository Function)
```bash
# Deploy NRF - must be first as other NFs register with it
helm install oai-nrf oai/oai-nrf -n oai-gnbsim \
  --set nrf.image.tag=v2.0.1 \
  --set nrf.config.logLevel=info

# Verify NRF is running
kubectl get pods -n oai-gnbsim | grep nrf
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep nrf | awk '{print $1}')
```

#### Deploy UDM (Unified Data Management)
```bash
# Deploy UDM
helm install oai-udm oai/oai-udm -n oai-gnbsim \
  --set udm.image.tag=v2.0.1 \
  --set udm.config.nrfUri="http://oai-nrf:80" \
  --set udm.config.logLevel=info

# Verify UDM deployment
kubectl get pods -n oai-gnbsim | grep udm
```

#### Deploy UDR (Unified Data Repository)
```bash
# Deploy UDR
helm install oai-udr oai/oai-udr -n oai-gnbsim \
  --set udr.image.tag=v2.0.1 \
  --set udr.config.nrfUri="http://oai-nrf:80" \
  --set udr.config.mysqlServer="mysql" \
  --set udr.config.logLevel=info

# Verify UDR deployment
kubectl get pods -n oai-gnbsim | grep udr
```

#### Deploy AUSF (Authentication Server Function)
```bash
# Deploy AUSF
helm install oai-ausf oai/oai-ausf -n oai-gnbsim \
  --set ausf.image.tag=v2.0.1 \
  --set ausf.config.nrfUri="http://oai-nrf:80" \
  --set ausf.config.logLevel=info

# Verify AUSF deployment
kubectl get pods -n oai-gnbsim | grep ausf
```

#### Deploy AMF (Access and Mobility Management Function)
```bash
# Deploy AMF
helm install oai-amf oai/oai-amf -n oai-gnbsim \
  --set amf.image.tag=v2.0.1 \
  --set amf.config.nrfUri="http://oai-nrf:80" \
  --set amf.config.logLevel=info \
  --set amf.config.guami.plmnId.mcc="208" \
  --set amf.config.guami.plmnId.mnc="95" \
  --set amf.config.servedGuamiList[0].plmnId.mcc="208" \
  --set amf.config.servedGuamiList[0].plmnId.mnc="95"

# Verify AMF deployment
kubectl get pods -n oai-gnbsim | grep amf
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep amf | awk '{print $1}')
```

#### Deploy SMF (Session Management Function)
```bash
# Deploy SMF
helm install oai-smf oai/oai-smf -n oai-gnbsim \
  --set smf.image.tag=v2.0.1 \
  --set smf.config.nrfUri="http://oai-nrf:80" \
  --set smf.config.logLevel=info \
  --set smf.config.upfInfo.upfIpAddress="oai-upf"

# Verify SMF deployment
kubectl get pods -n oai-gnbsim | grep smf
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep smf | awk '{print $1}')
```

#### Deploy UPF (User Plane Function)
```bash
# Deploy UPF
helm install oai-upf oai/oai-upf -n oai-gnbsim \
  --set upf.image.tag=v2.0.1 \
  --set upf.config.nrfUri="http://oai-nrf:80" \
  --set upf.config.logLevel=info \
  --set upf.config.pfcpAddress="oai-upf" \
  --set upf.config.gtpuAddress="oai-upf"

# Verify UPF deployment
kubectl get pods -n oai-gnbsim | grep upf
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep upf | awk '{print $1}')
```

### B.4: Build/Get gNBsim Image

#### Option B1: Build from source
```bash
# Clone gnbsim repository
cd ~
git clone https://gitlab.eurecom.fr/kharade/gnbsim.git
cd gnbsim

# Build Docker image
docker build --tag gnbsim:latest --target gnbsim --file docker/Dockerfile.ubuntu.22.04 .

# Tag for your container registry (if using)
docker tag gnbsim:latest your-registry/gnbsim:latest
docker push your-registry/gnbsim:latest
```

#### Option B2: Use pre-built image
```bash
docker pull rohankharade/gnbsim
docker image tag rohankharade/gnbsim:latest gnbsim:latest
```

### B.5: Deploy gNBsim

#### Create gNBsim Helm Values File
```bash
# Create values file for gNBsim
cat > gnbsim-values.yaml << EOF
gnbsim:
  image:
    repository: gnbsim
    tag: latest
    pullPolicy: IfNotPresent
  
  config:
    # gNB Configuration
    gnbId: 1
    gnbName: "gnbsim-001"
    mcc: "208"
    mnc: "95"
    tac: 1
    sst: 1
    
    # AMF Configuration
    amfAddress: "oai-amf"
    amfPort: 38412
    
    # UE Configuration
    numberOfUEs: 1
    imsiPrefix: "208950000000"
    imsiStart: 31
    
    # Network Configuration
    dataNetworkName: "default"
    
    # Logging
    logLevel: "info"

  service:
    type: ClusterIP
    ports:
      - name: ngap
        port: 38412
        targetPort: 38412
        protocol: SCTP
      - name: gtpu
        port: 2152
        targetPort: 2152
        protocol: UDP

  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
EOF
```

#### Deploy gNBsim using Helm
```bash
# Create gNBsim deployment
helm install oai-gnbsim ./gnbsim-chart -n oai-gnbsim -f gnbsim-values.yaml

# OR if using OAI official chart (when available)
helm install oai-gnbsim oai/oai-gnbsim -n oai-gnbsim \
  --set gnbsim.config.amfAddress="oai-amf" \
  --set gnbsim.config.numberOfUEs=1 \
  --set gnbsim.config.logLevel=info \
  --set gnbsim.config.mcc="208" \
  --set gnbsim.config.mnc="95"

# Verify gNBsim deployment
kubectl get pods -n oai-gnbsim | grep gnbsim
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep gnbsim | awk '{print $1}')
```

### B.6: Validate All Deployments

```bash
# Check all pods are running
kubectl get pods -n oai-gnbsim

# Check services
kubectl get svc -n oai-gnbsim

# Check resource utilization
kubectl top pods -n oai-gnbsim
```

### B.7: Validate UE Registration

```bash
# Check UE IP address allocation
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep gnbsim | awk '{print $1}') | grep "UE address:"

# Check AMF logs for UE registration
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep amf | awk '{print $1}') | grep -E "(5GMM-REGISTERED|UE.*registered)"

# Check gNBsim logs for connection status
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep gnbsim | awk '{print $1}') | grep -E "(connected|registered|established)"
```

### B.8: Connectivity Tests

#### Create Test Pod for Network Testing
```bash
# Create a test pod for network testing
kubectl run test-pod --image=ubuntu:20.04 -n oai-gnbsim --rm -it --restart=Never -- /bin/bash

# Inside the test pod, install network tools
apt update && apt install -y iputils-ping iperf3 curl

# Test connectivity to UE (if UE IP is 12.1.1.2)
ping -c 3 12.1.1.2

# Exit the test pod
exit
```

#### Port Forwarding for External Access
```bash
# Forward AMF port for external monitoring
kubectl port-forward -n oai-gnbsim svc/oai-amf 9090:9090 &

# Forward UPF port for testing
kubectl port-forward -n oai-gnbsim svc/oai-upf 8080:8080 &
```

### B.9: Scale Testing (Multiple UEs)

```bash
# Deploy additional gNBsim instances with different configurations
helm install oai-gnbsim2 ./gnbsim-chart -n oai-gnbsim \
  --set gnbsim.config.gnbId=2 \
  --set gnbsim.config.gnbName="gnbsim-002" \
  --set gnbsim.config.imsiStart=32 \
  --set gnbsim.config.numberOfUEs=1

helm install oai-gnbsim3 ./gnbsim-chart -n oai-gnbsim \
  --set gnbsim.config.gnbId=3 \
  --set gnbsim.config.gnbName="gnbsim-003" \
  --set gnbsim.config.imsiStart=33 \
  --set gnbsim.config.numberOfUEs=1

# Check all gNBsim instances
kubectl get pods -n oai-gnbsim | grep gnbsim

# Check UE registrations
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep gnbsim2 | awk '{print $1}') | grep "UE address:"
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep gnbsim3 | awk '{print $1}') | grep "UE address:"
```

### B.10: Log Collection

```bash
# Collect all logs using kubectl
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep mysql | awk '{print $1}') > /tmp/oai/mini-gnbsim/mysql.log 2>&1
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep nrf | awk '{print $1}') > /tmp/oai/mini-gnbsim/nrf.log 2>&1
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep udm | awk '{print $1}') > /tmp/oai/mini-gnbsim/udm.log 2>&1
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep udr | awk '{print $1}') > /tmp/oai/mini-gnbsim/udr.log 2>&1
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep ausf | awk '{print $1}') > /tmp/oai/mini-gnbsim/ausf.log 2>&1
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep amf | awk '{print $1}') > /tmp/oai/mini-gnbsim/amf.log 2>&1
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep smf | awk '{print $1}') > /tmp/oai/mini-gnbsim/smf.log 2>&1
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep upf | awk '{print $1}') > /tmp/oai/mini-gnbsim/upf.log 2>&1
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep gnbsim | awk '{print $1}') > /tmp/oai/mini-gnbsim/gnbsim.log 2>&1

# If multiple gNBsim instances exist
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep gnbsim2 | awk '{print $1}') > /tmp/oai/mini-gnbsim/gnbsim2.log 2>&1
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep gnbsim3 | awk '{print $1}') > /tmp/oai/mini-gnbsim/gnbsim3.log 2>&1
```

### B.11: Cleanup

```bash
# Clean up gNBsim deployments
helm uninstall oai-gnbsim -n oai-gnbsim
helm uninstall oai-gnbsim2 -n oai-gnbsim
helm uninstall oai-gnbsim3 -n oai-gnbsim

# Clean up 5G core network
helm uninstall oai-upf -n oai-gnbsim
helm uninstall oai-smf -n oai-gnbsim
helm uninstall oai-amf -n oai-gnbsim
helm uninstall oai-ausf -n oai-gnbsim
helm uninstall oai-udr -n oai-gnbsim
helm uninstall oai-udm -n oai-gnbsim
helm uninstall oai-nrf -n oai-gnbsim
helm uninstall mysql -n oai-gnbsim

# Delete namespace
kubectl delete namespace oai-gnbsim

# Verify cleanup
kubectl get namespaces | grep gnbsim
```

---

## Network IP Configuration

### Docker Compose IPs:
| Component | IP Address |
|-----------|------------|
| mysql | 192.168.70.131 |
| oai-amf | 192.168.70.132 |
| oai-smf | 192.168.70.133 |
| oai-upf | 192.168.70.134 |
| oai-ext-dn | 192.168.70.135 |
| gnbsim (gNB) | 192.168.70.136 |
| UE1 | 12.1.1.2 |
| UE2 | 12.1.1.3 |
| UE3 | 12.1.1.4 |
| UE4 | 12.1.1.5 |
| UE5 | 12.1.1.6 |

### Kubernetes Service Names:
| Component | Service Name | Port |
|-----------|-------------|------|
| MySQL | mysql | 3306 |
| NRF | oai-nrf | 80 |
| UDM | oai-udm | 80 |
| UDR | oai-udr | 80 |
| AUSF | oai-ausf | 80 |
| AMF | oai-amf | 80, 38412 |
| SMF | oai-smf | 80, 8080 |
| UPF | oai-upf | 8080, 2152 |
| gNBsim | oai-gnbsim | 38412, 2152 |

---

## Key Configuration Files

### Docker Compose Files:
- `docker-compose-gnbsim.yaml` - gNBsim configuration
- `core-network.py` - Core network deployment script

### Kubernetes Files:
- `gnbsim-values.yaml` - Helm values for gNBsim
- Individual Helm charts from OAI repository

---

## Troubleshooting

### Common Issues:

#### Docker Compose:
1. **Containers not healthy**: Check logs for configuration errors
2. **UE registration failed**: Verify AMF and database configuration
3. **No IP allocation**: Check SMF and UPF logs
4. **Connectivity issues**: Verify network routing and firewall rules

#### Kubernetes:
1. **Pods not starting**: Check resource limits and node capacity
2. **Service discovery issues**: Verify service names and DNS
3. **Persistent volume issues**: Check storage class and PV claims
4. **Network policies**: Verify pod-to-pod communication

### Debug Commands:

#### Docker Compose:
```bash
# Check container health
docker ps -a

# View real-time logs
docker logs -f <container_name>

# Check network connectivity
docker exec <container> ping <target_ip>
```

#### Kubernetes:
```bash
# Check pod status
kubectl get pods -n oai-gnbsim

# Describe pod for detailed info
kubectl describe pod <pod_name> -n oai-gnbsim

# View real-time logs
kubectl logs -f <pod_name> -n oai-gnbsim

# Check service endpoints
kubectl get endpoints -n oai-gnbsim

# Test network connectivity
kubectl exec -it <pod_name> -n oai-gnbsim -- ping <target_service>
```

---

## Comparison: Docker Compose vs Kubernetes

### Docker Compose Advantages:
- ✅ Simple setup and configuration
- ✅ Fast deployment and testing
- ✅ Easy debugging and log access
- ✅ Follows official OAI tutorial exactly
- ✅ Minimal resource overhead

### Kubernetes Advantages:
- ✅ Production-ready scalability
- ✅ Built-in service discovery
- ✅ Resource management and limits
- ✅ Rolling updates and rollbacks
- ✅ Integration with monitoring tools
- ✅ Multi-node deployment capability

### When to Use Which:
- **Docker Compose**: Learning, development, quick testing
- **Kubernetes**: Production deployment, scaling, research infrastructure

---

## Limitations (Critical for Research)

⚠️ **gNBsim Limitations (Both Deployment Methods):**
- No E2AP interface support
- Cannot integrate with FlexRIC
- No AI/ML RAN optimization capabilities
- Limited to basic 5G core testing
- Not suitable for advanced research

**For FlexRIC/AI-ML research → Use RFSim or hardware gNB**

---

## Advanced Kubernetes Configurations

### Persistent Storage for Database

```yaml
# mysql-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  namespace: oai-gnbsim
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/mysql-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: oai-gnbsim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```bash
# Apply persistent storage
kubectl apply -f mysql-pv.yaml

# Deploy MySQL with persistent storage
helm install mysql oai/mysql -n oai-gnbsim \
  --set mysql.persistence.enabled=true \
  --set mysql.persistence.existingClaim=mysql-pvc
```

### ConfigMaps for gNBsim Configuration

```yaml
# gnbsim-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gnbsim-config
  namespace: oai-gnbsim
data:
  config.yaml: |
    gnb:
      id: 1
      name: "gnbsim-001"
      mcc: "208"
      mnc: "95"
      tac: 1
      sst: 1
      sd: "0x123456"
    
    amf:
      address: "oai-amf"
      port: 38412
      
    ue:
      count: 1
      imsi_prefix: "208950000000"
      imsi_start: 31
      key: "465B5CE8B199B49FAA5F0A2EE238A6BC"
      opc: "E8ED289DEBA952E4283B54E88E6183CA"
      
    network:
      data_network_name: "default"
      
    logging:
      level: "info"
      output: "stdout"
```

```bash
# Apply ConfigMap
kubectl apply -f gnbsim-config.yaml

# Reference in Helm deployment
helm install oai-gnbsim ./gnbsim-chart -n oai-gnbsim \
  --set gnbsim.configMap.name=gnbsim-config
```

### Network Policies for Security

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: oai-gnbsim-policy
  namespace: oai-gnbsim
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 3306
    - protocol: SCTP
      port: 38412
    - protocol: UDP
      port: 2152
  egress:
  - to: []
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 3306
    - protocol: SCTP
      port: 38412
    - protocol: UDP
      port: 2152
    - protocol: UDP
      port: 53
```

```bash
# Apply network policy
kubectl apply -f network-policy.yaml
```

### Monitoring and Observability

#### Prometheus Configuration
```yaml
# prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: oai-gnbsim
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
    - job_name: 'oai-5g-core'
      static_configs:
      - targets: ['oai-amf:9090', 'oai-smf:8080', 'oai-upf:8080']
      metrics_path: /metrics
      scrape_interval: 30s
```

#### Deploy Prometheus
```bash
# Add prometheus helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Deploy Prometheus
helm install prometheus prometheus-community/prometheus -n oai-gnbsim \
  --set server.configMap.name=prometheus-config
```

#### Grafana Dashboard
```bash
# Deploy Grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm install grafana grafana/grafana -n oai-gnbsim \
  --set adminPassword=admin123 \
  --set service.type=NodePort

# Get Grafana admin password
kubectl get secret --namespace oai-gnbsim grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

### Horizontal Pod Autoscaler

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: gnbsim-hpa
  namespace: oai-gnbsim
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: oai-gnbsim
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
# Apply HPA
kubectl apply -f hpa.yaml

# Check HPA status
kubectl get hpa -n oai-gnbsim
```

### Advanced gNBsim Helm Chart

```yaml
# gnbsim-chart/values.yaml
replicaCount: 1

image:
  repository: gnbsim
  tag: latest
  pullPolicy: IfNotPresent

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}

podSecurityContext:
  fsGroup: 2000

securityContext:
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000

service:
  type: ClusterIP
  ports:
    ngap:
      port: 38412
      targetPort: 38412
      protocol: SCTP
    gtpu:
      port: 2152
      targetPort: 2152
      protocol: UDP

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

nodeSelector: {}

tolerations: []

affinity: {}

gnbsim:
  config:
    gnb:
      id: 1
      name: "gnbsim-001"
      mcc: "208"
      mnc: "95"
      tac: 1
      sst: 1
      sd: "0x123456"
    
    amf:
      address: "oai-amf"
      port: 38412
      
    ue:
      count: 1
      imsi_prefix: "208950000000"
      imsi_start: 31
      
    network:
      data_network_name: "default"
      
    logging:
      level: "info"

  persistence:
    enabled: false
    storageClass: ""
    accessMode: ReadWriteOnce
    size: 1Gi

  configMap:
    create: true
    name: ""

  networkPolicy:
    enabled: false
    
  monitoring:
    enabled: false
    port: 9090
    path: /metrics
```

```yaml
# gnbsim-chart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "gnbsim.fullname" . }}
  labels:
    {{- include "gnbsim.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "gnbsim.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "gnbsim.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "gnbsim.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: ngap
              containerPort: 38412
              protocol: SCTP
            - name: gtpu
              containerPort: 2152
              protocol: UDP
          {{- if .Values.gnbsim.monitoring.enabled }}
            - name: metrics
              containerPort: {{ .Values.gnbsim.monitoring.port }}
              protocol: TCP
          {{- end }}
          env:
            - name: AMF_ADDRESS
              value: {{ .Values.gnbsim.config.amf.address }}
            - name: AMF_PORT
              value: "{{ .Values.gnbsim.config.amf.port }}"
            - name: GNB_ID
              value: "{{ .Values.gnbsim.config.gnb.id }}"
            - name: MCC
              value: {{ .Values.gnbsim.config.gnb.mcc }}
            - name: MNC
              value: {{ .Values.gnbsim.config.gnb.mnc }}
            - name: UE_COUNT
              value: "{{ .Values.gnbsim.config.ue.count }}"
            - name: LOG_LEVEL
              value: {{ .Values.gnbsim.config.logging.level }}
          {{- if .Values.gnbsim.configMap.create }}
          volumeMounts:
            - name: config-volume
              mountPath: /etc/gnbsim
              readOnly: true
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- if .Values.gnbsim.configMap.create }}
      volumes:
        - name: config-volume
          configMap:
            name: {{ include "gnbsim.configMapName" . }}
      {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### Load Testing with Multiple gNBsim Instances

```bash
# Create load test script
cat > load-test.sh << 'EOF'
#!/bin/bash

NAMESPACE="oai-gnbsim"
BASE_NAME="gnbsim-load"
NUM_INSTANCES=50
CONCURRENT_DEPLOYS=10

echo "Starting load test with $NUM_INSTANCES gNBsim instances"

# Function to deploy gNBsim instance
deploy_gnbsim() {
    local instance_id=$1
    local gnb_id=$((instance_id + 1000))
    local imsi_start=$((31 + instance_id))
    
    helm install ${BASE_NAME}-${instance_id} ./gnbsim-chart -n $NAMESPACE \
        --set gnbsim.config.gnb.id=$gnb_id \
        --set gnbsim.config.gnb.name="gnbsim-load-${instance_id}" \
        --set gnbsim.config.ue.imsi_start=$imsi_start \
        --set gnbsim.config.ue.count=1 \
        --set resources.requests.cpu=100m \
        --set resources.requests.memory=256Mi \
        --set resources.limits.cpu=500m \
        --set resources.limits.memory=512Mi \
        --wait --timeout=300s
    
    echo "Deployed ${BASE_NAME}-${instance_id} with gNB ID $gnb_id"
}

# Deploy instances in parallel batches
for ((i=0; i<$NUM_INSTANCES; i+=$CONCURRENT_DEPLOYS)); do
    echo "Deploying batch starting at instance $i"
    
    # Deploy concurrent instances
    for ((j=0; j<$CONCURRENT_DEPLOYS && (i+j)<$NUM_INSTANCES; j++)); do
        deploy_gnbsim $((i+j)) &
    done
    
    # Wait for current batch to complete
    wait
    
    # Check cluster health
    echo "Checking cluster health after batch $((i/$CONCURRENT_DEPLOYS + 1))"
    kubectl top nodes
    kubectl get pods -n $NAMESPACE | grep -c "Running"
    
    # Brief pause between batches
    sleep 30
done

echo "Load test deployment completed"
echo "Total pods: $(kubectl get pods -n $NAMESPACE | grep -c gnbsim)"
echo "Registered UEs: $(kubectl logs -n $NAMESPACE -l app=gnbsim --tail=100 | grep -c "UE address:")"
EOF

chmod +x load-test.sh
./load-test.sh
```

### Performance Monitoring Script

```bash
# Create monitoring script
cat > monitor-performance.sh << 'EOF'
#!/bin/bash

NAMESPACE="oai-gnbsim"
DURATION=300  # 5 minutes
INTERVAL=30   # 30 seconds

echo "Starting performance monitoring for $DURATION seconds"

for ((i=0; i<$DURATION; i+=$INTERVAL)); do
    echo "=== Monitoring Report: $(date) ==="
    
    # Cluster resource usage
    echo "Node Resources:"
    kubectl top nodes
    
    # Pod resource usage
    echo "Pod Resources (Top 10):"
    kubectl top pods -n $NAMESPACE | head -10
    
    # Pod status summary
    echo "Pod Status Summary:"
    kubectl get pods -n $NAMESPACE --no-headers | awk '{print $3}' | sort | uniq -c
    
    # Network metrics
    echo "Network Metrics:"
    AMF_POD=$(kubectl get pods -n $NAMESPACE -l app=oai-amf -o jsonpath='{.items[0].metadata.name}')
    kubectl logs -n $NAMESPACE $AMF_POD --tail=10 | grep -E "(registered|connected|established)" | tail -5
    
    # UE registration count
    echo "UE Registration Count:"
    kubectl logs -n $NAMESPACE -l app=gnbsim --tail=1000 | grep -c "UE address:" || echo "0"
    
    echo "================================"
    echo ""
    
    sleep $INTERVAL
done

echo "Performance monitoring completed"
EOF

chmod +x monitor-performance.sh
./monitor-performance.sh
```

### Cleanup Script for Load Testing

```bash
# Create cleanup script
cat > cleanup-load-test.sh << 'EOF'
#!/bin/bash

NAMESPACE="oai-gnbsim"
BASE_NAME="gnbsim-load"

echo "Cleaning up load test deployments..."

# Get all load test releases
RELEASES=$(helm list -n $NAMESPACE -q | grep "^${BASE_NAME}-")

if [ -z "$RELEASES" ]; then
    echo "No load test releases found"
    exit 0
fi

echo "Found $(echo "$RELEASES" | wc -l) load test releases"

# Uninstall releases in parallel
echo "$RELEASES" | xargs -n 1 -P 10 -I {} helm uninstall {} -n $NAMESPACE

echo "Waiting for pods to terminate..."
kubectl wait --for=delete pod -l app=gnbsim-load -n $NAMESPACE --timeout=300s

echo "Load test cleanup completed"
kubectl get pods -n $NAMESPACE | grep gnbsim || echo "No gnbsim pods remaining"
EOF

chmod +x cleanup-load-test.sh
# Run when needed: ./cleanup-load-test.sh
```

---

## Production Deployment Considerations

### Security Hardening

```yaml
# security-policies.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gnbsim-sa
  namespace: oai-gnbsim
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gnbsim-role
  namespace: oai-gnbsim
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gnbsim-binding
  namespace: oai-gnbsim
subjects:
- kind: ServiceAccount
  name: gnbsim-sa
  namespace: oai-gnbsim
roleRef:
  kind: Role
  name: gnbsim-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: policy/v1
kind: PodSecurityPolicy
metadata:
  name: gnbsim-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

### Resource Quotas

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: oai-gnbsim-quota
  namespace: oai-gnbsim
spec:
  hard:
    requests.cpu: "50"
    requests.memory: 100Gi
    limits.cpu: "100"
    limits.memory: 200Gi
    count/pods: 200
    count/services: 50
    count/configmaps: 50
    count/secrets: 50
    count/persistentvolumeclaims: 20
```

### Backup and Recovery

```bash
# Create backup script
cat > backup-oai.sh << 'EOF'
#!/bin/bash

NAMESPACE="oai-gnbsim"
BACKUP_DIR="/tmp/oai-backup/$(date +%Y%m%d-%H%M%S)"
mkdir -p $BACKUP_DIR

echo "Creating backup in $BACKUP_DIR"

# Backup Helm releases
helm list -n $NAMESPACE -o yaml > $BACKUP_DIR/helm-releases.yaml

# Backup ConfigMaps
kubectl get configmaps -n $NAMESPACE -o yaml > $BACKUP_DIR/configmaps.yaml

# Backup Secrets
kubectl get secrets -n $NAMESPACE -o yaml > $BACKUP_DIR/secrets.yaml

# Backup PVCs
kubectl get pvc -n $NAMESPACE -o yaml > $BACKUP_DIR/pvcs.yaml

# Backup database (if using external MySQL)
MYSQL_POD=$(kubectl get pods -n $NAMESPACE -l app=mysql -o jsonpath='{.items[0].metadata.name}')
if [ ! -z "$MYSQL_POD" ]; then
    kubectl exec -n $NAMESPACE $MYSQL_POD -- mysqldump -u root -plinux oai_db > $BACKUP_DIR/database.sql
fi

echo "Backup completed: $BACKUP_DIR"
tar -czf $BACKUP_DIR.tar.gz -C $(dirname $BACKUP_DIR) $(basename $BACKUP_DIR)
echo "Backup archived: $BACKUP_DIR.tar.gz"
EOF

chmod +x backup-oai.sh
```

---

## Your Specific Workflow Integration

Based on your notes, here's the exact workflow you've been using:

### Complete Kubernetes Deployment Script

```bash
#!/bin/bash
# Based on your notes - complete workflow script

# Set environment variables
export OAI_CN5G_FED_DIR=~/oai-cn5g-fed
export NAMESPACE=oai-gnbsim-new

echo "=== Starting OAI 5G Core + gNBsim Deployment ==="

# 1. Environment Setup
echo "Setting up Kubernetes environment..."
kubectl version
kubectl get namespaces
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
kubectl config set-context --current --namespace=$NAMESPACE

# Verify namespace
echo "Current namespace: $(kubectl config view --minify | grep namespace)"

# 2. Clone repository if not exists
if [ ! -d "$OAI_CN5G_FED_DIR" ]; then
    echo "Cloning OAI CN5G Fed repository..."
    git clone https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed.git $OAI_CN5G_FED_DIR
fi

# 3. Navigate to charts directory
cd $OAI_CN5G_FED_DIR/charts/gnbsim-chart/charts/e2e_scenarios/case1

# 4. Update Helm dependencies
echo "Updating Helm dependencies..."
helm dependency update

# 5. Deploy or upgrade gNBsim
echo "Deploying gNBsim..."
if helm list -n $NAMESPACE | grep -q gnbsim; then
    echo "Upgrading existing gnbsim deployment..."
    helm upgrade gnbsim ./ -n $NAMESPACE
else
    echo "Installing new gnbsim deployment..."
    helm install gnbsim ./ -n $NAMESPACE
fi

# 6. Wait for pods to be ready
echo "Waiting for pods to be ready..."
kubectl wait --for=condition=ready pod --all -n $NAMESPACE --timeout=300s

# 7. Check pod status
echo "Checking pod status..."
kubectl get pods -n $NAMESPACE

# 8. Verify connectivity
echo "=== Network Connectivity Tests ==="

# Get pod names
GNB_POD=$(kubectl get pods -n $NAMESPACE -l app=oai-gnb -o jsonpath='{.items[0].metadata.name}')
UPF_POD=$(kubectl get pods -n $NAMESPACE -l app=oai-upf -o jsonpath='{.items[0].metadata.name}')

if [ ! -z "$GNB_POD" ] && [ ! -z "$UPF_POD" ]; then
    echo "gNB Pod: $GNB_POD"
    echo "UPF Pod: $UPF_POD"
    
    # DNS resolution test
    echo "Testing DNS resolution..."
    kubectl exec -it $GNB_POD -n $NAMESPACE -- getent hosts oai-upf
    
    # Get UPF IP and test connectivity
    UPF_IP=$(kubectl get svc oai-upf -n $NAMESPACE -o jsonpath='{.spec.clusterIP}')
    echo "UPF Service IP: $UPF_IP"
    
    echo "Testing ping connectivity..."
    kubectl exec -it $GNB_POD -n $NAMESPACE -- ping -c 4 $UPF_IP
    
    echo "Starting GTP-U traffic capture (Ctrl+C to stop)..."
    echo "kubectl exec -it $UPF_POD -n $NAMESPACE -c tcpdump -- tcpdump -i any udp port 2152"
else
    echo "Could not find gNB or UPF pods. Check deployment."
fi

echo "=== Deployment Complete ==="
```

### Quick Commands Reference (Your Notes)

```bash
# Environment setup
kubectl version
kubectl get namespaces
kubectl config set-context --current --namespace=oai-gnbsim-new
kubectl config view --minify | grep namespace

# Repository and deployment
git clone https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed.git
cd ~/oai-cn5g-fed/charts/gnbsim-chart/charts/e2e_scenarios/case1
helm dependency update

# Deploy/upgrade
helm install gnbsim ./ -n oai-gnbsim-new    # New installation
helm upgrade gnbsim ./ -n oai-gnbsim-new    # Upgrade existing

# Status check
kubectl get pods -n oai-gnbsim-new

# Network tests
kubectl exec -it oai-gnb-87d48996c-gwnh4 -n oai-gnbsim-new -- getent hosts oai-upf
kubectl exec -it oai-gnb-87d48996c-gwnh4 -n oai-gnbsim-new -- ping -c 4 172.16.77.165
kubectl exec -it oai-upf-569845989b-vkxqf -n oai-gnbsim-new -c tcpdump -- tcpdump -i any udp port 2152
```

### Docker Compose Alternative (Your Method)

```bash
# Environment setup
export OAI_CN5G_FED_DIR=~/oai-cn5g-fed
cd $OAI_CN5G_FED_DIR/docker-compose

# Core network deployment
docker-compose -f docker-compose-basic-nrf-ebpf.yaml up -d
# OR
python3 ./core-network.py --type start-basic

# gNBsim deployment
docker-compose -f docker-compose-gnbsim.yaml up -d gnbsim
```

---

## Key Insights from Your Notes

### 1. **Working E2E Scenario Structure**
Your approach using `charts/gnbsim-chart/charts/e2e_scenarios/case1` is excellent because:
- It's a **complete end-to-end scenario**
- All dependencies are properly configured
- Network connectivity is pre-tested
- It includes all necessary components (AMF, SMF, UPF, gNB, UE)

### 2. **Network Connectivity Verification**
Your systematic approach to verify connectivity is spot-on:
- **DNS resolution**: Confirms service discovery works
- **IP connectivity**: Validates network routing
- **GTP-U traffic capture**: Monitors actual data plane traffic

### 3. **Pod-to-Pod Communication**
Your tests confirm the critical aspects:
- gNB can resolve UPF service name
- ICMP connectivity works between pods
- GTP-U tunneling is functioning (port 2152)

### 4. **Troubleshooting Approach**
Your method of using `tcpdump` inside the UPF pod is excellent for:
- Monitoring GTP-U encapsulated traffic
- Verifying data plane functionality
- Debugging network issues

---

## Additional Commands Based on Your Workflow

### Enhanced Network Debugging

```bash
# Get detailed pod information
kubectl describe pod $GNB_POD -n oai-gnbsim-new
kubectl describe pod $UPF_POD -n oai-gnbsim-new

# Check service endpoints
kubectl get endpoints -n oai-gnbsim-new

# Monitor logs in real-time
kubectl logs -f $GNB_POD -n oai-gnbsim-new
kubectl logs -f $UPF_POD -n oai-gnbsim-new

# Check network policies
kubectl get networkpolicies -n oai-gnbsim-new

# Test service connectivity
kubectl exec -it $GNB_POD -n oai-gnbsim-new -- nslookup oai-upf
kubectl exec -it $GNB_POD -n oai-gnbsim-new -- telnet oai-upf 8080
```

### Performance Monitoring

```bash
# Check resource usage
kubectl top pods -n oai-gnbsim-new
kubectl top nodes

# Monitor network traffic
kubectl exec -it $UPF_POD -n oai-gnbsim-new -c tcpdump -- tcpdump -i any -n 'port 2152 or port 8080'

# Check for packet drops
kubectl exec -it $UPF_POD -n oai-gnbsim-new -- netstat -s | grep -i drop
```

The E2E scenario approach is particularly valuable for ensuring all components work together properly.2>&1 | grep "UE address:"
docker logs gnbsim4 2>&1 | grep "UE address:"
docker logs gnbsim5 2>&1 | grep "UE address:"

# Verify all UEs are registered
docker logs oai-amf | tail -50
```

## Step 8: Log Collection

```bash
# Collect all logs
docker logs oai-amf > /tmp/oai/mini-gnbsim/amf.log 2>&1
docker logs oai-smf > /tmp/oai/mini-gnbsim/smf.log 2>&1
docker logs oai-upf > /tmp/oai/mini-gnbsim/upf.log 2>&1
docker logs oai-ext-dn > /tmp/oai/mini-gnbsim/ext-dn.log 2>&1
docker logs gnbsim > /tmp/oai/mini-gnbsim/gnbsim.log 2>&1
docker logs gnbsim2 > /tmp/oai/mini-gnbsim/gnbsim2.log 2>&1
docker logs gnbsim3 > /tmp/oai/mini-gnbsim/gnbsim3.log 2>&1
docker logs gnbsim4 > /tmp/oai/mini-gnbsim/gnbsim4.log 2>&1
docker logs gnbsim5 > /tmp/oai/mini-gnbsim/gnbsim5.log 2>&1
```

## Step 9: Cleanup

```bash
# Remove gNBsim containers
docker-compose -f docker-compose-gnbsim.yaml down -t 0

# Remove 5G core network
python3 ./core-network.py --type stop-mini --scenario 2
```

## Network IP Configuration

| Component | IP Address |
|-----------|------------|
| mysql | 192.168.70.131 |
| oai-amf | 192.168.70.132 |
| oai-smf | 192.168.70.133 |
| oai-upf | 192.168.70.134 |
| oai-ext-dn | 192.168.70.135 |
| gnbsim (gNB) | 192.168.70.136 |
| UE1 | 12.1.1.2 |
| UE2 | 12.1.1.3 |
| UE3 | 12.1.1.4 |
| UE4 | 12.1.1.5 |
| UE5 | 12.1.1.6 |

## Key Configuration Files

The tutorial references these files in the `docker-compose` directory:
- `docker-compose-gnbsim.yaml` - gNBsim configuration
- `core-network.py` - Core network deployment script
- Database contains pre-configured IMSIs: 208950000000031-208950000000040

## Troubleshooting

### Common Issues:
1. **Containers not healthy**: Check logs for configuration errors
2. **UE registration failed**: Verify AMF and database configuration
3. **No IP allocation**: Check SMF and UPF logs
4. **Connectivity issues**: Verify network routing and firewall rules

### Debug Commands:
```bash
# Check container health
docker ps -a

# View real-time logs
docker logs -f <container_name>

# Check network connectivity
docker exec <container> ping <target_ip>
```

## Limitations (Critical for Research)

⚠️ **gNBsim Limitations:**
- No E2AP interface support
- Cannot integrate with FlexRIC
- No AI/ML RAN optimization capabilities
- Limited to basic 5G core testing
- Not suitable for advanced research

**For FlexRIC/AI-ML research → Use RFSim or hardware gNB**
