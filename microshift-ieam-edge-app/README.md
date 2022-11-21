# Deploy a sample edge application on Microshift cluster and manage it with IBM Edge Application Manager (IEAM).

The step-by-step guidance for preparing the IBM edge device environment and deploying a sample edge application on Microshift managed by IBM Edge Application Manager 
(IEAM).

## Navigation

- [Prerequisites](#prerequisites)
- [Deploy image registry](#deploy-image-registry)
- [Install IEAM agent](#install-ieam-agent)
- [Deploy services to edge cluster](#deploy-services-to-edge-cluster)

## Prerequisites

1. Have Microshift deployed on the edge device either via [microshift-rhacm](https://github.com/gargpriyank/telco-edge/tree/main/microshift-rhacm) or 
 [microshift-raspberry-pi](https://github.com/gargpriyank/telco-edge/tree/main/microshift-raspberry-pi).
2. Have [IEAM Management Hub](https://www.ibm.com/docs/en/eam/4.4?topic=installation-install-ieam) installed on the managed from Red Hat OpenShift cluster. Collect 
   the necessary files for IEAM agent installation
   1. Login to the edge device as root user. Use `oc` command to login to the Red Hat OpenShift cluster where IEAM is deployed. Ensure all pods in "ibm-common-services" 
      and "ibm-edge" namespaces are either Running or Completed
      ```markdown
      oc project ibm-edge
      oc get pods -n ibm-common-services
      oc get pods
      ```
   2. Log in, pull and extract the agent bundle with your entitlement key through the [Entitled Registry](https://myibm.ibm.com/products-services/containerlibrary)
      ```markdown
      podman login cp.icr.io --username cp && \
      podman rm -f ibm-eam-4.4.0-bundle; \
      podman pull cp.icr.io/cp/ieam/ibm-eam-bundle:2.29.0-638 && \
      podman create --name ibm-eam-4.4.0-bundle cp.icr.io/cp/ieam/ibm-eam-bundle:2.29.0-638 bash && \
      podman cp ibm-eam-4.4.0-bundle:/ibm-eam-4.4.0-bundle.tar.gz ibm-eam-4.4.0-bundle.tar.gz && \
      tar -zxvf ibm-eam-4.4.0-bundle.tar.gz && \
      cd ~/ibm-eam-4.4.0-bundle/tools
      ```
   3. Validate the installation state
      ```markdown
      ./service_healthcheck.sh
      Output:
      ==Running service verification tests for IBM Edge Application Manager==
      SUCCESS: IBM Edge Application Manager Exchange API is operational
      SUCCESS: IBM Edge Application Manager Cloud Sync Service is operational
      SUCCESS: IBM Edge Application Manager Agbot database heartbeat is current
      SUCCESS: IBM Edge Application Manager SDO API is operational
      SUCCESS: IBM Edge Application Manager UI is properly requiring valid authentication
      ==All expected services are up and running==
      ```
   4. Install the hzn CLI. Use "ppc64le" or "x86_64" directory depending on the hardware architecture
      ```markdown
      cd ~/ibm-eam-4.4.0-bundle/agent && \
      tar -zxvf edge-packages-4.4.0.tar.gz
      
      rpm-ostree install make git jq
      rpm-ostree install ./edge-packages-4.4.0/linux/rpm/x86_64/horizon-cli-*.x86_64.rpm
      systemctl reboot
      ```
   5. Replace "docker" with "podman". Create first "Org" in IEAM for multi-tenancy
      ```markdown
      cd ~/ibm-eam-4.4.0-bundle/tools
      sed -i 's/docker/podman/gi' post_install.sh
      ./post_install.sh <choose-your-org-name>
      ```
   6. Set IEAM cluster URL, user credentials and `podman` authentication environment variables
      ```markdown
      cd ~/ibm-eam-4.4.0-bundle
      export CLUSTER_URL=https://$(oc get cm management-ingress-ibmcloud-cluster-info -o jsonpath='{.data.cluster_ca_domain}' -n ibm-common-services)
      oc --insecure-skip-tls-verify=true -n kube-public get secret ibmcloud-cluster-ca-cert -o jsonpath="{.data.ca\.crt}" | base64 --decode > ieam.crt
      export HZN_MGMT_HUB_CERT_PATH="$PWD/ieam.crt"
      export HZN_FSS_CSSURL=${CLUSTER_URL}/edge-css
      export CLUSTER_USERNAME=$(oc -n ibm-common-services get secret platform-auth-idp-credentials -o jsonpath='{.data.admin_username}' | base64 --decode)
      export CLUSTER_USERPASS=$(oc -n ibm-common-services get secret platform-auth-idp-credentials -o jsonpath='{.data.admin_password}' | base64 --decode)
      export REGISTRY_USERNAME=cp
      export REGISTRY_PASSWORD=<ENTITLEMENT_KEY>
      ```
   7. Download and install IBM Cloud Pak CLI [cloudctl](https://www.ibm.com/docs/en/eam/4.4?topic=cli-installing-cloudctl-kubectl-oc) binary. Use `cloudctl` to login to 
      IEAM management hub
      ```markdown
      cloudctl login -a $CLUSTER_URL -u $CLUSTER_USERNAME -p $CLUSTER_USERPASS -n ibm-edge --skip-ssl-validation
      ```
   8. Generate the files for edge device installation
      ```markdown
      cd ~/ibm-eam-4.4.0-bundle/agent
      cp /usr/bin/edgeNodeFiles.sh edgeNodeFiles.sh
      sed -i 's/docker/podman/g' ./edgeNodeFiles.sh
      sed -i 's/podman.io/docker.io/g' ./edgeNodeFiles.sh
      HZN_EXCHANGE_USER_AUTH='' ./edgeNodeFiles.sh ALL -c -p edge-packages-4.4.0 -r cp.icr.io/cp/ieam
      ```
   9. Create API key. Find the key value in the output and save it for future use
      ```markdown
      cloudctl iam api-key-create "<choose-an-api-key-name>" -d "<choose-an-api-key-description>"
      ```
      
## Deploy image registry

1. Replace "kubeconfig" to connect back to microshift cluster and check the storage class name. It should be `kubevirt-hostpath-provisioner`
   ```markdown
   cat /var/lib/microshift/resources/kubeadmin/kubeconfig > ~/.kube/config
   oc get storageclasses
   ```
2. Use `image-registry.yaml` provided with this repo to deploy image registry
   ```markdown
   mkdir /var/registry /var/hpvolumes
   chmod 777 /var/registry
   chcon -t container_file_t -R /var/hpvolumes
   systemctl reboot
   ls -lZ /var
   oc apply -f image-registry.yaml
   ```
3. Verify the image registry is deployed
   ```markdown
   oc get pods -n image-registry
   ```
   
## Install IEAM agent

1. Set environment variables
   ```markdown
   export HZN_EXCHANGE_USER_AUTH=iamapikey:<api-key>
   export HZN_ORG_ID=<your-exchange-organization>
   export HZN_FSS_CSSURL=https://<ieam-management-hub-ingress>/edge-css/
   export EDGE_CLUSTER_STORAGE_CLASS=kubevirt-hostpath-provisioner
   export REGISTRY_ENDPOINT=$(oc get service image-registry -n image-registry  -o 'jsonpath={.spec.clusterIP}'):5000
   export IMAGE_ON_EDGE_CLUSTER_REGISTRY=${REGISTRY_ENDPOINT}/openhorizon-agent/amd64_anax_k8s
   ```
2. Update "registries.conf" to add the image registry endpoint, restart crio and microshift services
   ```markdown
   cat << EOF | tee -a /etc/containers/registries.conf
   > [[registry]]
   > location = "${REGISTRY_ENDPOINT}"
   > insecure = true
   > EOF
   
   systemctl restart crio microshift
   ```
3. Download `agent-install.sh` script from the Cloud Sync Service (CSS)
   ```markdown
   cd ~/ibm-eam-4.4.0-bundle/agent
   curl -u "$HZN_ORG_ID/$HZN_EXCHANGE_USER_AUTH" -k -o agent-install.sh $HZN_FSS_CSSURL/api/v1/objects/IBM/agent_files/agent-install.sh/data
   chmod +x agent-install.sh
   ```
4. Install edge agent on the microshift cluster
   ```markdown
   ./agent-install.sh -D cluster -i 'css:'
   ```
5. (Optional) For uninstalling edge agent in case of an error
   ```markdown
   ./agent-uninstall.sh -u $HZN_EXCHANGE_USER_AUTH -d
   ```
   
## Deploy services to edge cluster

1. The hzn command is inside the agent container, set the aliases to run hzn from the host
   ```markdown
   cat << 'END_ALIASES' >> ~/.bash_aliases
   > alias getagentpod='kubectl -n openhorizon-agent get pods --selector=app=agent -o jsonpath={.items[].metadata.name}'
   > alias hzn='kubectl -n openhorizon-agent exec -i $(getagentpod) -- hzn'
   > END_ALIASES
   
   source ~/.bash_aliases
   ```
2. Verify edge node is configured
   ```markdown
   hzn node list
   ```
3. Set the node policy to deploy helloworld operator and service
   ```markdown
   mkdir ~/ieam-apps
   cd ~/ieam-apps
   cat << 'EOF' > operator-example-node.policy.json
   > {
   > "properties": [
   > { "name": "openhorizon.example", "value": "nginx-operator" }
   > ]
   > }
   > EOF
   
   cat operator-example-node.policy.json | hzn policy update -f-
   hzn policy list
   ```
4. After a minute, check for an agreement and the running edge operator and service containers
   ```markdown
   kubectl -n openhorizon-agent get pods
   hzn agreement list
   ```