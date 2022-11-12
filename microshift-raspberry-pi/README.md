# Deploy Microshift on Raspberry Pi.

The step-by-step guidance for deploying Microshift on Raspberry Pi.

## Navigation

- [Initial Preparation](#initial-preparation)
- [Installation](#installation)
- [Import Microshift cluster in RHACM](#import-microshift-cluster-in-rhacm)
- [Deploy a simple application to Microshift cluster from RHACM console](#deploy-a-simple-application-to-microshift-cluster-from-rhacm-console)

## Initial Preparation

Download the [Fedora IOT](https://getfedora.org/en/iot/download/) 36 raw aarch64 image and create [Bootable SD Card](https://docs.fedoraproject.
org/en-US/iot/physical-device-setup/) 
for your Raspberry Pi.
   
## Installation

1. Boot your Raspberry Pi using the SD card and let the Fedora OS configured and installed. Login as root user. Configure Wifi
   ```markdown
   nmcli device wifi list
   nmcli device wifi connect <SSID|BSSID> password <password>
   ```
2. Configure RPM repos
   ```markdown
   curl -L -o /etc/yum.repos.d/fedora-modular.repo https://src.fedoraproject.org/rpms/fedora-repos/raw/rawhide/f/fedora-modular.repo
   curl -L -o /etc/yum.repos.d/fedora-updates-modular.repo https://src.fedoraproject.org/rpms/fedora-repos/raw/rawhide/f/fedora-updates-modular.repo
   curl -L -o /etc/yum.repos.d/group_redhat-et-microshift-fedora-36.repo https://copr.fedorainfracloud.org/coprs/g/redhat-et/microshift/repo/fedora-36/group_redhat-et-microshift-fedora-36.repo
   ```
3. Install and setup dependencies and CRI-O
   ```markdown
   rpm-ostree install cri-o cri-tools
   ```
4. [podman](https://podman.io/) will already be installed. Search the `auth.json` in the root directory, if not there create one and replace the content
with your [pull secret](https://cloud.redhat.com/openshift/install/pull-secret). Use podman to log-in to registry.
   ```markdown
   podman login registry.redhat.io --tls-verify=false --authfile <authfile_path>
   ```
5. Install Microshift
   ```markdown
   rpm-ostree install microshift
   ```
6. Enable firewall
   ```markdown
   firewall-cmd --zone=trusted --add-source=10.42.0.0/16 --permanent
   firewall-cmd --zone=public --add-port=80/tcp --permanent
   firewall-cmd --zone=public --add-port=443/tcp --permanent
   firewall-cmd --zone=public --add-port=5353/udp --permanent
   firewall-cmd --reload
   ```
7. Start Microshift service
   ```markdown
   systemctl reboot
   curl -L https://github.com/openshift/microshift/releases/download/nightly/microshift-linux-arm64 > /usr/local/bin/microshift
   chmod +x /usr/local/bin/microshift
   cp /usr/lib/systemd/system/microshift.service /etc/systemd/system/microshift.service
   sed -i "s|/usr/bin|/usr/local/bin|" /etc/systemd/system/microshift.service
   systemctl daemon-reload
   systemctl enable crio microshift --now
   ``` 
8. Setup oc and kubectl CLI
   ```markdown
   curl -O https://mirror.openshift.com/pub/openshift-v4/arm64/clients/ocp/stable/openshift-client-linux.tar.gz
   tar -xf openshift-client-linux.tar.gz -C /usr/local/bin oc kubectl
   ```
9. Configure kubeconfig
   ```markdown
   mkdir ~/.kube
   cat /var/lib/microshift/resources/kubeadmin/kubeconfig > ~/.kube/config
   ```
10. Verify installation
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