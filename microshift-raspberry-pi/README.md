# Deploy Microshift on Raspberry Pi.

The step-by-step guidance for deploying Microshift on Raspberry Pi.

## Navigation

- [Initial Preparation](#initial-preparation)
- [Installation](#installation)
- [Import Microshift cluster in RHACM](#import-microshift-cluster-in-rhacm)
- [Deploy a simple application to Microshift cluster from RHACM console](#deploy-a-simple-application-to-microshift-cluster-from-rhacm-console)

## Initial Preparation

Download the [Fedora Minimal](https://arm.fedoraproject.org/) raw image and use the [Raspberry Pi Imager](https://www.raspberrypi.com/software/) 
to upload the image and install the OS on your SD card that will be used as a boot device for your Raspberry Pi.
   
## Installation

1. Boot your Raspberry Pi using the SD card and let the Fedora OS configured and installed. Login as root user. Configure Wifi
   ```markdown
   nmcli device wifi list
   nmcli device wifi connect <SSID|BSSID> password <password>
   ```
2. Install and setup dependencies and CRI-O
   ```markdown
   dnf -y update
   dnf -y install git golang rpm-build selinux-policy-devel container-selinux
   dnf -y module enable cri-o:1.21
   dnf -y install cri-o cri-tools
   ```
3. Reboot the device and enable CRI-O
   ```markdown
   systemctl reboot
   systemctl enable crio --now
   ```
4. Verify CRI-O using crictl
   ```markdown
   crictl info
   ```
5. Install [podman](https://podman.io/). Search the `auth.json` in the root directory, if not there create one and replace the content 
with your [pull secret](https://cloud.redhat.com/openshift/install/pull-secret). Use podman to log-in to registry
   ```markdown
   dnf -y install podman
   podman login registry.redhat.io --tls-verify=false --authfile <authfile_path>
   ```
6. Install Microshift
   ```markdown
   dnf -y copr enable @redhat-et/microshift fedora-36-aarch64
   ```
   For hardware architecture mismatch issue, edit the repo and replace basearch with aarch64 and then install Microshift with forced architecture.
   ```markdown
   vi /etc/yum.repos.d/_copr:copr.fedorainfracloud.org:group_redhat-et:microshift.repo
   dnf -y install microshift --forcearch aarch64
   ```
8. Enable firewall
   ```markdown
   firewall-cmd --zone=trusted --add-source=10.42.0.0/16 --permanent
   firewall-cmd --zone=public --add-port=80/tcp --permanent
   firewall-cmd --zone=public --add-port=443/tcp --permanent
   firewall-cmd --zone=public --add-port=5353/udp --permanent
   firewall-cmd --reload
   ```
9. Start Microshift service
   ```markdown
   systemctl enable microshift --now
   ``` 
10. Setup oc and kubectl CLI
    ```markdown
    curl -O https://mirror.openshift.com/pub/openshift-v4/arm64/clients/ocp/stable/openshift-client-linux.tar.gz
    tar -xf openshift-client-linux.tar.gz -C /usr/local/bin oc kubectl
    ```
11. Configure kubeconfig
    ```markdown
    mkdir ~/.kube
    cat /var/lib/microshift/resources/kubeadmin/kubeconfig > ~/.kube/config
    ```
12. Verify installation
     ```markdown
     oc get pods -A
   
     NAMESPACE                       NAME                                  READY   STATUS    RESTARTS   AGE
     kube-system                     kube-flannel-ds-kbztk                 1/1     Running   0          10m
     kubevirt-hostpath-provisioner   kubevirt-hostpath-provisioner-4f28f   1/1     Running   0          6m29s
     openshift-dns                   dns-default-w4vnj                     2/2     Running   0          10m
     openshift-dns                   node-resolver-4zlgk                   1/1     Running   0          10m
     openshift-ingress               router-default-6c96f6bc66-94gfd       1/1     Running   0          10m
     openshift-service-ca            service-ca-7bffb6f6bf-kvvkd           1/1     Running   0          10m
     ```

## Import Microshift cluster in RHACM

1. Log-in to your managed from RHOCP cluster and open the RHACM console from the routes.
2. Select the Clusters tab from the left pane and select Import cluster.
3. On the Import an existing cluster screen, provide a valid cluster name, cluster set (if any) and select import mode as Run import commands manually.
4. This will generate a command. Copy and run this command on your device where Microshift cluster is installed.
5. A new namespace `open-cluster-management-agent` will be created. Wait until all the pods are in `Running` state.
6. Go back to RHACM console, you should see your Microshift cluster in `Ready` state.

## Deploy a simple application to Microshift cluster from RHACM console

1. In the RHACM console, select the Application tab from the left pane.
2. Click Create application and select Subscription from the drop-down menu.
3. Provide a name, namespace, git URL ([sample applications](https://github.com/stolostron/application-samples)) and the cluster (Label, Value) that should match with 
your Microshift cluster label.
4. Hit Create button, that should deploy the sample application onto the Microshift cluster.