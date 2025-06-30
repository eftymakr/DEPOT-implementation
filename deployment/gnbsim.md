# Phase 1: gNBsim Deployment (Initial Attempt)

## Overview
This documents our initial deployment attempt using gNBsim, following the standard [OAI gNBsim tutorial](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed/-/blob/master/docs/DEPLOY_SA5G_MINI_WITH_GNBSIM.md).

## Why We Started Here
- **Standard OAI approach**: Well-documented tutorial
- **Simplicity**: Basic gNB simulation without complex RF modeling
- **Core network testing**: Good for validating 5G core functions
- **Learning purpose**: Understanding basic 5G SA deployment

## Prerequisites
```bash
# Verify Kubernetes cluster
kubectl cluster-info
kubectl get nodes

# Check available resources
kubectl top nodes
```

## Deployment Commands

### 1. Environment Setup
```bash
# Create namespace for gNBsim deployment
kubectl create namespace oai-gnbsim

# Verify namespace creation
kubectl get namespaces | grep gnbsim
```

### 2. Helm Repository Setup
```bash
# Add official OAI Helm repository
helm repo add oai https://charts.oai-ran.org/

# Update repository information
helm repo update

# List available OAI charts
helm search repo oai
```

### 3. Core Network Deployment

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

#### Deploy AMF (Access and Mobility Management Function)
```bash
# Deploy AMF
helm install oai-amf oai/oai-amf -n oai-gnbsim \
  --set amf.image.tag=v2.0.1 \
  --set amf.config.nrfUri="http://oai-nrf:80" \
  --set amf.config.logLevel=info

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
  --set smf.config.logLevel=info

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
  --set upf.config.logLevel=info

# Verify UPF deployment
kubectl get pods -n oai-gnbsim | grep upf
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep upf | awk '{print $1}')
```

### 4. Deploy gNBsim
```bash
# Deploy gNBsim (simulated gNB + UE)
helm install oai-gnbsim oai/oai-gnbsim -n oai-gnbsim \
  --set gnbsim.config.amfAddress="oai-amf" \
  --set gnbsim.config.numberOfUEs=1 \
  --set gnbsim.config.logLevel=info

# Verify gNBsim deployment
kubectl get pods -n oai-gnbsim | grep gnbsim
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep gnbsim | awk '{print $1}')
```

## Validation Commands

### Check All Pods
```bash
# Verify all components are running
kubectl get pods -n oai-gnbsim

# Check resource utilization
kubectl top pods -n oai-gnbsim
```

### Test UE Registration
```bash
# Check AMF logs for UE registration
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep amf | awk '{print $1}') \
  | grep -E "(5GMM-REGISTERED|UE.*registered)"

# Check gNBsim logs for connection status
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep gnbsim | awk '{print $1}') \
  | grep -E "(connected|registered|established)"
```

## Results Achieved

### ✅ What Worked
- **Core network deployment**: All 5G core functions (NRF, AMF, SMF, UPF) deployed successfully
- **Basic UE registration**: gNBsim could register UEs with the core network
- **Simple connectivity**: Basic data plane connectivity through simulated gNB
- **Learning value**: Good for understanding 5G core network interactions

### ⚠️ Limitations Discovered

#### 1. No E2AP Interface Support
```bash
# Check gNBsim logs for E2AP - will find nothing
kubectl logs -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep gnbsim | awk '{print $1}') \
  | grep -i "e2ap"
# No E2AP interface logs found
```

#### 2. FlexRIC Incompatibility
- **Missing E2 Application Protocol**: gNBsim lacks E2AP interface
- **No RAN Intelligence**: Cannot connect to RAN Intelligence Controller (RIC)
- **Limited AI/ML capabilities**: No support for AI/ML-driven network optimization
- **Research limitations**: Insufficient for advanced 5G research

#### 3. Basic Simulation Only
- **Simple traffic patterns**: Limited to basic UE simulation
- **No advanced features**: Missing advanced RAN functionalities
- **Limited metrics**: Basic performance monitoring only

## Critical Discovery

> **Key Finding**: FlexRIC requires E2AP interface for AI/ML RAN optimization, which gNBsim doesn't provide.

### E2AP Interface Investigation
```bash
# Search for E2AP capability in gNBsim
kubectl exec -n oai-gnbsim $(kubectl get pods -n oai-gnbsim | grep gnbsim | awk '{print $1}') \
  -- find /opt -name "*e2*" -o -name "*E2*"
# Result: No E2AP-related files found

# Check gNBsim configuration options
kubectl get configmap -n oai-gnbsim
kubectl describe configmap <gnbsim-config> -n oai-gnbsim
# Result: No E2AP configuration options available
```

## Research Requirements vs. gNBsim Capabilities

| Requirement | gNBsim Support | Status |
|-------------|----------------|---------|
| Basic 5G Core Testing | ✅ Yes | Works |
| UE Registration | ✅ Yes | Works |
| Data Connectivity | ✅ Yes | Basic |
| E2AP Interface | ❌ No | **Missing** |
| FlexRIC Integration | ❌ No | **Impossible** |
| AI/ML RAN Optimization | ❌ No | **Cannot achieve** |
| Advanced Research | ❌ Limited | **Insufficient** |

## Decision Point: Transition Required

### Research Goals vs. Current Capabilities
**Research Goal**: AI/ML-driven 5G network optimization using FlexRIC
**gNBsim Reality**: No E2AP support = No FlexRIC integration possible

### Solution Analysis
1. **Stay with gNBsim**: Limited to basic 5G testing, no research value
2. **Switch to RFSim**: Enables E2AP interface, supports FlexRIC integration
3. **Hardware deployment**: Too complex and expensive for initial research

**Decision**: Switch to RFSim deployment for E2AP compatibility.

## Cleanup Commands
```bash
# Clean up gNBsim deployment (run these before Phase 2)
helm uninstall oai-gnbsim -n oai-gnbsim
helm uninstall oai-upf -n oai-gnbsim
helm uninstall oai-smf -n oai-gnbsim
helm uninstall oai-amf -n oai-gnbsim
helm uninstall oai-nrf -n oai-gnbsim

# Delete namespace
kubectl delete namespace oai-gnbsim

# Verify cleanup
kubectl get namespaces | grep gnbsim
```

## Lessons Learned

### Technical Insights
1. **Interface requirements matter**: E2AP is essential for FlexRIC
2. **Simulation limitations**: gNBsim is for basic testing only
3. **Research planning**: Need to verify tool capabilities before deployment
4. **Architecture decisions**: Tool choice impacts research possibilities

### Next Steps
➡️ **Phase 2**: Deploy RFSim for E2AP interface support
➡️ **Enable FlexRIC**: Integrate RAN Intelligence Controller
➡️ **Research applications**: Develop AI/ML optimization xApps

## References
- [OAI gNBsim Tutorial](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed/-/blob/master/docs/DEPLOY_SA5G_MINI_WITH_GNBSIM.md)
- [FlexRIC E2AP Requirements](https://gitlab.eurecom.fr/mosaic5g/flexric)
- [E2AP Interface Documentation](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/openair2/E2AP/README.md)

---

**Status**: ✅ Deployment Successful | ❌ E2AP Missing | ➡️ **Transition to RFSim Required**
