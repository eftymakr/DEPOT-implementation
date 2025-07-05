# REFERENCES

## Official OpenAirInterface Documentation

### Core Resources
- **OAI GitLab Repository**: https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed
- **OAI 5G Core Documentation**: https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed/-/blob/master/docs/DEPLOY_SA5G_HC.md
- **OAI Tutorials**: https://gitlab.eurecom.fr/oai/openairinterface5g/-/wikis/tutorials
- **OAI Docker Hub**: https://hub.docker.com/u/oaisoftwarealliance

### Technical Specifications
- **3GPP TS 38.300**: NR and NG-RAN Overall Description
- **3GPP TS 23.501**: 5G System Architecture and Procedures  
- **3GPP TS 38.331**: Radio Resource Control (RRC) Protocol
- **3GPP TS 38.211**: Physical Channels and Modulation

## Kubernetes and Container Orchestration

### Kubernetes Documentation
- **Kubernetes Official Docs**: https://kubernetes.io/docs/
- **kubectl Command Reference**: https://kubernetes.io/docs/reference/kubectl/
- **Pod Networking**: https://kubernetes.io/docs/concepts/cluster-administration/networking/

### Helm Documentation
- **Helm Official Docs**: https://helm.sh/docs/
- **Helm Charts Best Practices**: https://helm.sh/docs/chart_best_practices/
- **Managing Dependencies**: https://helm.sh/docs/helm/helm_dependency/

### Container Networking
- **Calico Documentation**: https://docs.tigera.io/calico/latest/about/
- **Multus CNI**: https://github.com/k8snetworkplumbingwg/multus-cni
- **Container Network Interface (CNI)**: https://github.com/containernetworking/cni

## 5G Technology References

### 5G Architecture
- **5G Architecture Overview**: https://www.3gpp.org/technologies/5g-system-overview
- **Service Based Architecture**: https://www.3gpp.org/news-events/2178-sa_5g
- **Network Slicing**: https://www.3gpp.org/news-events/2073-nsa

### RAN Technologies
- **RF Simulator**: https://gitlab.eurecom.fr/oai/openairinterface5g/-/wikis/how-to-build#rf-simulator
- **E2 Interface Specification**: https://www.o-ran.org/specifications
- **O-RAN Alliance**: https://www.o-ran.org/

## FlexRIC and xApps

### FlexRIC Framework
- **FlexRIC Repository**: https://gitlab.eurecom.fr/mosaic5g/flexric
- **FlexRIC Documentation**: https://gitlab.eurecom.fr/mosaic5g/flexric/-/blob/main/README.md
- **Near-RT RIC**: https://docs.o-ran-sc.org/en/latest/

### xApp Development
- **xApp Development Guide**: https://docs.o-ran-sc.org/projects/o-ran-sc-it-dev/en/latest/
- **E2 Service Model**: https://docs.o-ran-sc.org/en/latest/submodules/ric-plt-e2/docs/
- **RIC Platform**: https://wiki.o-ran-sc.org/display/RICP

## Research Papers and Academic References

### 5G Core Network Studies
- *"Performance Analysis of 5G Core Network"* - IEEE Communications Magazine
- *"Containerized 5G Core Network Deployment"* - IEEE Network
- *"Cloud-Native 5G Networks"* - Computer Networks Journal

### RF Simulation and Testing
- *"RF Channel Modeling for 5G Networks"* - IEEE Transactions on Wireless Communications
- *"Software-Defined Radio for 5G Research"* - IEEE Communications Surveys & Tutorials

### RAN Intelligence and AI/ML
- *"Artificial Intelligence for Radio Access Networks"* - IEEE Communications Magazine
- *"Machine Learning in 5G RAN Optimization"* - IEEE Network
- *"E2E Network Slicing with AI/ML"* - Computer Communications

## Tools and Utilities

### Monitoring and Debugging
- **Wireshark 5G Dissectors**: https://wiki.wireshark.org/5G
- **kubectl Cheat Sheet**: https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- **Docker Logging**: https://docs.docker.com/config/containers/logging/

### Development Tools
- **Visual Studio Code**: https://code.visualstudio.com/
- **Git Documentation**: https://git-scm.com/doc
- **Markdown Guide**: https://www.markdownguide.org/

## Community Resources

### Forums and Support
- **OAI Discourse Forum**: https://openairinterface.discourse.group/
- **Stack Overflow - 5G Tags**: https://stackoverflow.com/questions/tagged/5g
- **Kubernetes Community**: https://kubernetes.io/community/

### Conferences and Events
- **Mobile World Congress (MWC)**: https://www.mwcbarcelona.com/
- **IEEE Globecom**: https://globecom2024.ieee-globecom.org/
- **O-RAN Alliance Events**: https://www.o-ran.org/events

## Version Information

### Software Versions Used
- **Kubernetes**: v1.27.4+
- **Helm**: v3.11.2+
- **OAI 5G Core**: v2.0.1
- **Docker**: 20.10+
- **Calico CNI**: v3.26+

### Image Tags
```yaml
# Core Network Functions
oai-amf: v2.0.1
oai-smf: v2.0.1  
oai-upf: v2.0.1
oai-nrf: v2.0.1
oai-udm: v2.0.1
oai-udr: v2.0.1
oai-ausf: v2.0.1

# RAN Functions  
oai-gnb: develop
oai-nr-ue: develop
```

## License and Citation

### License Information
- **OAI License**: Apache License 2.0
- **Repository License**: [ License]


```

## Acknowledgments

- **OpenAirInterface Software Alliance** for providing the open-source 5G implementation
- **EURECOM** for OAI development and documentation
- **O-RAN Alliance** for E2 interface specifications
- **Kubernetes Community** for container orchestration platform
- **3GPP** for 5G standards and specifications

---

*Last Updated: July 2025*  
*This reference list supports the DEPOT 5G implementation project*
