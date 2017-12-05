# Installing contrail-kubernetes objects only with an existing contrail cluster

#### Prerequisites

1. Ubuntu 16.04.2 host for k8s clusters
2. Kubernetes cluster should be up and running
3. Firewall service needs to be disabled
    ```console
    sudo service ufw stop
    sudo iptables -F
    ```
4. Patch Liveness Probe and Readiness Probe

    ```console
    kubectl patch deploy/kube-dns --type json  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/readinessProbe", "value": {"exec": {"command": ["wget", "-O", "-", "http://127.0.0.1:8081/readiness"]}}}]' -n kube-system
    kubectl patch deploy/kube-dns --type json  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/livenessProbe", "value": {"exec": {"command": ["wget", "-O", "-", "http://127.0.0.1:10054/healthcheck/kubedns"]}}}]' -n kube-system && kubectl patch deploy/kube-dns --type json  -p='[{"op": "replace", "path": "/spec/template/spec/containers/1/livenessProbe", "value": {"exec": {"command": ["wget", "-O", "-", "http://127.0.0.1:10054/healthcheck/dnsmasq"]}}}]' -n kube-system && kubectl patch deploy/kube-dns --type json  -p='[{"op": "replace", "path": "/spec/template/spec/containers/2/livenessProbe", "value": {"exec": {"command": ["wget", "-O", "-", "http://127.0.0.1:10054/metrics"]}}}]' -n kube-system
    ```
5. Make sure to delete below old folders from previous contrail installation

    ```console
    sudo rm -rf /var/lib/contrail*
    sudo rm -rf /var/lib/configdb*
    sudo rm -rf /var/lib/analyticsdb*
    ```

#### Provision contrail CNI and contrail-agent

1. Get the source code

    ```console
    git clone https://github.com/madhukar32/contrail-k8s-openstack.git
    ```

2. Edit variables in configmap and change it according to your setup. Please refer to a sample configmap data below

    ```yaml
    data:
      global-config: |-
        [GLOBAL]
        cloud_orchestrator = kubernetes
        config_nodes = 10.87.65.159
        controller_nodes = 10.87.65.159
        analytics_nodes = 10.87.65.159
        kubemanager-config: |-
          [KUBERNETES]
          cluster_name = k8s-default
          cluster_project = {}
          cluster_network = {'domain': 'default-domain', 'project': 'admin', 'name': 'k8s_default_vn'}
          service_subnets = 10.96.0.0/12
          pod_subnets = 10.32.0.0/12
          api_server = 10.87.65.156
          [AUTH]
          ip = 10.87.65.159
          admin_password = contrail123
          admin_user = admin
          admin_tenant = admin
        kubernetes-agent-config: |-
          [AGENT]
    ```
  In the above data
  * config_nodes, controller_nodes and analytics_nodes refer to a comma seperated list of IP address of your contrail cluster
  * cluster_network, refers to the FQDN of the kubernetes cluster network
  * cluster_project, if cluster_network is given then the project name will be derived from it
  * service_subnets and pod_subnets, CIDR of service and pod subnet respectively
  * api_server, IP address of the k8s master
  * AUTH, section where you will give the keystone auth details

3. Deploy contrail objects
    ```console
    kubectl apply -f contrail-k8s-openstack.yaml
    ```

4. Verifying contrail-status
  ```bash
  for pod_name in `kubectl get pods -n kube-system -o wide | grep contrail | awk '{print $1}'`; do kubectl exec -it  $pod_name -n kube-system -- contrail-status ; done
  ```
