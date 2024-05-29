# free5gc-kubernetes

This repository contains the necessary files and resources to deploy and operate Free5GC, an open-source 5G core network implementation. It provides Kubernetes manifest files for deploying Free5GC using microservices, and Free5GC WebUI. Additionally, there are manifest files for deploying the MongoDB database and network attachment definitions for Free5GC.

For more information about Free5GC, please visit the [Free5GC GitHub repository](https://github.com/free5gc/free5gc).

![Static Badge](https://img.shields.io/badge/stable-v1.0.0-green)
![Static Badge](https://img.shields.io/badge/free5gc-v3.4.1-green)
![Static Badge](https://img.shields.io/badge/ueransim-v3.4.1-green)
![Static Badge](https://img.shields.io/badge/k8s-v1.28.10-green)
![Static Badge](https://img.shields.io/badge/kernel-v5.4.0-green) 

## (Opcional) Autocompletion and alias k for kubectl in Bash.

```sh
# Enable command completion for kubectl in Bash.
echo 'source <(kubectl completion bash)' >>~/.bashrc
# Create an alias k that you can use instead of typing kubectl each time.
echo 'alias k=kubectl' >>~/.bashrc
# Set up command completion for the alias k using the same completion function as kubectl.
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
# Reload the ~/.bashrc file to apply the changes immediately.
source ~/.bashrc
```

## Deployment

**Note**: Free5GC recommends kernel version `5.4.0`. It has also been successfully tested on kernel `5.15.0-94`. Using a higher kernel version (e.g, 6.x) may result in [issues with the UPF](https://forum.free5gc.org/t/upf-est-createfar-error-invalid-argument/2111). 

**Note**: The deployment instructions assume a working kubernetes cluster with OVS CNI installed. You can optionally use the [testbed-automator](https://github.com/niloysh/testbed-automator) to prepare the Kubernetes cluster. This includes setting up the K8s cluster, configuring the cluster, installing various Container Network Interfaces (CNIs), configuring OVS bridges, and preparing for the deployment of the 5G Core network.

To deploy Free5GC and its components, follow the deployment steps below:

1. Set up OVS bridges. On each K8s cluster node, add the OVS bridges: n2br, n3br, and n4br. Connect nodes using these bridges and OVS-based VXLAN tunnels. See [ovs-cni docs](https://github.com/k8snetworkplumbingwg/ovs-cni/blob/main/docs/demo.md#connect-bridges-using-vxlan).

    <details>
    <summary>Example command for creating VXLAN tunnels</summary>

    ```bash
    sudo ovs-vsctl add-port n2br vxlan_nuc1_n2 -- set Interface vxlan_nuc1_n2 type=vxlan options:remote_ip=<remote_ip> options:key=1002
    ```
    </details>  

<br>

**Note**: The testbed-automator scripts automatically configures the OVS bridges for a single-node cluster setup, and VXLAN tunnel creation is not required. For a multi-node cluster configuration, VXLAN tunnels are used for node interconnectivity.

2. Create test namespace using `kubectl create ns test`.

3. (optional) Create a **test** context.

    ```sh
    kubectl config view # See the kubectl config.
    kubectl config current-context # See the actual context.
    kubectl config set-context <context-name> --namespace=<namespace-name> --cluster=<cluster-name> --user=<user-name>  # Create a new context.
    kubectl config use-context <context-name> # Use the context created.
    ```

4. Deploy the MongoDB database using the Kubernetes manifest files provided in the mongodb/ directory. See deploying components. Wait for the db pod to be in the Running state before proceeding to the next step.

5. Configuring bridges for use by ovs-cni.

    ```sh
    sudo ovs-vsctl --may-exist add-br n2br
    sudo ovs-vsctl --may-exist add-br n3br
    sudo ovs-vsctl --may-exist add-br n4br
    sudo ovs-vsctl --may-exist add-br n5br
    sudo ovs-vsctl --may-exist add-br n6br
    ```

6. Deploy the network attachment definitions using manifest files in the networks5g/ directory. This are used for the secondary interfaces of the UPF, SMF, etc.

7. Install the gtp5g kernel module for Free5GC. Install gtp5g v0.8.9 on nodes where UPF should run. This is a prerequisite for deploying the UPF.

    ```sh
    git clone -b v0.8.9 https://github.com/free5gc/gtp5g.git
    cd gtp5g
    make clean && make
    sudo make install
    ```

**Note:** To unistall gtp5g run `sudo make uninstall`.

8. Deploy Free5GC using the Kubernetes manifest files in the free5gc/ directory. The pods should eventually be in the Running state. You need deploy the NFs in a certain order. Use `kubectl apply -f <nf>/` command to deploy the NFs. The order is: upf, db, nrf, amf, ausf, nssf, pcf, smf, udm, udr. 

    **FALTAN:** webui, chf, ue.

9. 

