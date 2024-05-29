# free5gc-kubernetes

This repository contains the necessary files and resources to deploy and operate Free5GC, an open-source 5G core network implementation. It provides Kubernetes manifest files for deploying Free5GC using microservices, and Free5GC WebUI. Additionally, there are manifest files for deploying the MongoDB database and network attachment definitions for Free5GC.

For more information about Free5GC, please visit the [Free5GC GitHub repository](https://github.com/free5gc/free5gc).

![Static Badge](https://img.shields.io/badge/stable-v1.0.0-green)
![Static Badge](https://img.shields.io/badge/free5gc-v3.4.1-green)
![Static Badge](https://img.shields.io/badge/ueransim-v3.4.1-green)
![Static Badge](https://img.shields.io/badge/k8s-v1.28.10-green)
![Static Badge](https://img.shields.io/badge/ubuntu-v20.04_LTS-green) 
![Static Badge](https://img.shields.io/badge/kernel-v5.4.0-green) 

## (Opcional) Autocompletion to k alias in Bash.

```sh
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

7. Install the gtp5g kernel module for Free5GC.

    ```sh
    cd bin
    ./install-gtp5g.sh
    ```

    To unistall gtp5g:
    
    ```sh
    cd build/gtp5g
    sudo make uninstall
    ```

8. Deploy Free5GC using the Kubernetes manifest files in the free5gc/ directory. The pods should eventually be in the Running state. You need deploy the NFs in a certain order. Use `kubectl apply -f <nf>/` command to deploy the NFs. The order is: upf, nrf, amf, ausf, nssf, pcf, smf, udm, udr, webui, chf, n3iwf, n3iwue. 

9. The ueransim directory contains Kubernetes manifest files for both gNB and UEs. First, deploy UERANSIM gNB using ueransim/ueransim-gnb directory and wait for NGAP connection to succeed. You should see the following in the gNB log.

10. Ensure correct UE subscriber information is inserted. You can enter subscription information using the web UI. Subscribers can be added using the Free5GC WebUI. The WebUI is accessible at `http://<node-ip>:30530`. The default username and password are admin and free5gc, respectively. Subscriber details can be found in UE config files (e.g., ue1.yaml).

11. Deploy UERANSIM UE using ueransim/ueransim-ue/ directory. Once the UE is connected, you should see the following logs:

    ```sh
    tesis@tesis-PC:~/tesis/5g-monarch2/free5gc-k8s-test/free5gc$ k logs ue-68979c6bd9-v9mbx 
    UERANSIM v3.2.6
    [2024-05-29 20:18:14.365] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
    [2024-05-29 20:18:14.367] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
    [2024-05-29 20:18:14.372] [nas] [info] Selected plmn[208/93]
    [2024-05-29 20:18:14.372] [rrc] [info] Selected cell plmn[208/93] tac[1] category[SUITABLE]
    [2024-05-29 20:18:14.372] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
    [2024-05-29 20:18:14.372] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
    [2024-05-29 20:18:14.372] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
    [2024-05-29 20:18:14.372] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
    [2024-05-29 20:18:14.372] [nas] [debug] Sending Initial Registration
    [2024-05-29 20:18:14.373] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
    [2024-05-29 20:18:14.373] [rrc] [debug] Sending RRC Setup Request
    [2024-05-29 20:18:14.378] [rrc] [info] RRC connection established
    [2024-05-29 20:18:14.378] [rrc] [info] UE switches to state [RRC-CONNECTED]
    [2024-05-29 20:18:14.378] [nas] [info] UE switches to state [CM-CONNECTED]
    [2024-05-29 20:18:14.757] [nas] [debug] Authentication Request received
    [2024-05-29 20:18:14.757] [nas] [debug] Received SQN [000000000023]
    [2024-05-29 20:18:14.757] [nas] [debug] SQN-MS [000000000000]
    [2024-05-29 20:18:14.802] [nas] [debug] Security Mode Command received
    [2024-05-29 20:18:14.802] [nas] [debug] Selected integrity[2] ciphering[0]
    [2024-05-29 20:18:14.940] [nas] [debug] Registration accept received
    [2024-05-29 20:18:14.940] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
    [2024-05-29 20:18:14.940] [nas] [debug] Sending Registration Complete
    [2024-05-29 20:18:14.940] [nas] [info] Initial Registration is successful
    [2024-05-29 20:18:14.940] [nas] [debug] Sending PDU Session Establishment Request
    [2024-05-29 20:18:14.941] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
    [2024-05-29 20:18:14.941] [nas] [debug] Sending PDU Session Establishment Request
    [2024-05-29 20:18:14.941] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
    [2024-05-29 20:18:15.147] [nas] [debug] Configuration Update Command received
    [2024-05-29 20:18:15.552] [nas] [debug] PDU Session Establishment Accept received
    [2024-05-29 20:18:15.552] [nas] [info] PDU Session establishment is successful PSI[1]
    [2024-05-29 20:18:15.562] [nas] [debug] PDU Session Establishment Accept received
    [2024-05-29 20:18:15.562] [nas] [info] PDU Session establishment is successful PSI[2]
    [2024-05-29 20:18:15.657] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
    [2024-05-29 20:18:15.847] [app] [info] Connection setup for PDU session[2] is successful, TUN interface[uesimtun1, 10.61.0.1] is up.
    ```




