---
title: Apiserver 增加IP
---

# 生成kubeadm配置
```bash
# kubectl -n kube-system get configmap kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' > kubeadm.yaml
```

# 修改kubeadm配置
在certSANs中增加IP
```yaml
..
apiServer:
  certSANs:
  - kubernetes
  - kubernetes.default
  - kubernetes.default.svc
  - kubernetes.default.svc.cluster.local
  - localhost
  - 127.0.0.1
  - lb.kubesphere.local
  - 192.168.3.30
  - devops
  - devops.cluster.local
  - 10.233.0.1
  - m.k8s.local
  ...
```



# 备份并生成证书
```
# mv /etc/kubernetes/pki/apiserver.{crt,key} ~/cert_bak
# kubeadm init phase certs apiserver --config kubeadm.yaml 
```

# 验证
```
# openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text
...
 X509v3 Subject Alternative Name: 
                DNS:devops, DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, DNS:lb.kubesphere.local, DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, DNS:localhost, DNS:lb.kubesphere.local, DNS:devops, DNS:devops.cluster.local, DNS:m.k8s.local, IP Address:10.233.0.1, IP Address:192.168.3.30, IP Address:127.0.0.1, IP Address:192.168.3.30, IP Address:10.233.0.1
...
```