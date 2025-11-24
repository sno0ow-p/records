---
title: CNI Plugin Cilium
date: 2025-05-20
pin: false
tags:
- k8s
- Network
- cni
- cilium
---

# CNI Plugin Cilium

## Concept

### Why Cilium

- 현대 마이크로서비스 아키텍쳐에서는 각기의 컨테이너가 요구사항에 따라 배포 혹은 셧다운이 자주 발생

  iptables와 같은 전통적인 방식은 매번 룰을 수정해야하기 때문에 오버헤드가 큼

- eBPF 기반

- 네트워크 커널 레벨에서의 Observability 확보 가능

## Routing

## IP Address Management

## eBPF Datapath

## Observability

## References

- https://isovalent.com/blog/post/why-replace-iptables-with-ebpf/
- https://isovalent.com/blog/post/cilium-netkit-a-new-container-networking-paradigm-for-the-ai-era/#h-roadblock-4-a-legacy-virtual-cable