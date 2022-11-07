# Deploy Microshift on Raspberry Pi.

The step-by-step guidance for deploying Microshift on Raspberry Pi.

## Navigation

- [Installation](#installation)
- [Initial Preparation](#initial-preparation)
- [Import Microshift cluster in RHACM](#import-microshift-cluster-in-rhacm)
- [Deploy a simple application to Microshift cluster from RHACM console](#deploy-a-simple-application-to-microshift-cluster-from-rhacm-console)

## Initial Preparation

Follow [Raspberry Pi OS installer](https://www.raspberrypi.com/software/) documentation to install Fedberry OS on your Raspberry Pi.
   
## Installation

1. Boot your Raspberry Pi using the SD card. Login as root user. Install and setup CRI-O
   ```markdown
   dnf -y install 'dnf-command(copr)'
   dnf -y copr enable rhcontainerbot/container-selinux
   curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/CentOS_8/devel:kubic:libcontainers:stable.repo
   curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:${VERSION}/CentOS_8/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo
   dnf install -y cri-o cri-tools
   rpm-ostree install cri-o cri-tools
   ```
2. Reboot the device and enable CRI-O
   ```markdown
   systemctl reboot
   systemctl enable crio --now
   ```
3. Verify CRI-O using crictl
   ```markdown
   crictl info
   ```
4. [podman](https://podman.io/) will already be installed. Search the `auth.json` in the root directory and replace the content 
with your [pull secret](https://cloud.redhat.com/openshift/install/pull-secret). Use podman to log-in to registry.
   ```markdown
   podman login registry.redhat.io --tls-verify=false --authfile <authfile_path>
   ```
6. Install Microshift
   ```markdown
   rpm-ostree install microshift
   ```
7. Enable firewall
   ```markdown
   firewall-cmd --zone=trusted --add-source=10.42.0.0/16 --permanent
   firewall-cmd --zone=public --add-port=80/tcp --permanent
   firewall-cmd --zone=public --add-port=443/tcp --permanent
   firewall-cmd --zone=public --add-port=5353/udp --permanent
   firewall-cmd --reload
   ```
8. Start Microshift service
   ```markdown
   systemctl enable microshift --now
   ``` 
9. Setup oc and kubectl CLI
   ```markdown
   curl -O https://mirror.openshift.com/pub/openshift-v4/$(uname -m)/clients/ocp/stable/openshift-client-linux.tar.gz
   tar -xf openshift-client-linux.tar.gz -C /usr/local/bin oc kubectl
   ```
10. Configure kubeconfig
    ```markdown
    mkdir ~/.kube
    sudo cat /var/lib/microshift/resources/kubeadmin/kubeconfig > ~/.kube/config
    ```
11. Verify installation
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