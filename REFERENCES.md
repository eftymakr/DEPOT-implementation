# REFERENCES

## Official OpenAirInterface Documentation

### Core Resources
- **OAI GitLab Repository**: https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed
- **OAI GNBSIM Documentation**: https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed/-/blob/master/docs/DEPLOY_SA5G_HC.md
- **OAI 5G Core Documentation**: https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed/-/blob/master/docs/DEPLOY_SA5G_HC.md
- **FLexRIC Documentation**: https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/openair2/E2AP/README.md
- **OAI Docker Hub**: https://hub.docker.com/u/oaisoftwarealliance


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
- **Service Based Architecture**: 
- **Network Slicing**:

### RAN Technologies
- **RF Simulator**: https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed/-/blob/master/docs/DEPLOY_SA5G_HC.md
- **O-RAN Alliance**: https://www.o-ran.org/

## FlexRIC and xApps

### FlexRIC Framework
- **FlexRIC Repository**: https://gitlab.eurecom.fr/mosaic5g/flexric
- **FLexRIC E2AP Documentation**: https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/openair2/E2AP/README.md
- **Near-RT RIC**: https://docs.o-ran-sc.org/en/latest/

### xApp Development
- **xApp Development Guide**: https://docs.o-ran-sc.org/projects/o-ran-sc-it-dev/en/latest/
- **E2 Service Model**: https://docs.o-ran-sc.org/en/latest/submodules/ric-plt-e2/docs/
- **RIC Platform**: https://wiki.o-ran-sc.org/display/RICP


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
