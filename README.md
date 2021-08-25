# Assisted-Installer-API
Using Assisted Installer with API - https://console.redhat.com/openshift/assisted-installer/clusters

references - https://cloudcult.dev/cilium-installation-openshift-assisted-installer/

Pull reference: https://github.com/rh-telco-tigers/Assisted-Installer-API

# Downloading Offline Token

1. Click [here](https://console.redhat.com/openshift/token) to get the token
   ![Openshift Console](https://github.com/rh-telco-tigers/Assisted-Installer-API/blob/main/images/load-token.png)

2. Click on Load Token
   ![Load Token](https://github.com/rh-telco-tigers/Assisted-Installer-API/blob/main/images/copy-token.png)

3. Use copy to clipboard function and provide it as a variable OFFLINE_ACCESS_TOKEN
   ```bash
   OFFLINE_ACCESS_TOKEN="<PASTE_TOKEN_HERE>"

   export TOKEN=$(curl \
   --silent \
   --data-urlencode "grant_type=refresh_token" \
   --data-urlencode "client_id=cloud-services" \
   --data-urlencode "refresh_token=${OFFLINE_ACCESS_TOKEN}" \
   https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token | \
   jq -r .access_token)
   ```
   
4. Create a file (cluster-details) with the following content. Modfy this as per your environment. 
    ```bash
    export ASSISTED_SERVICE_API="api.openshift.com"
    export CLUSTER_VERSION="4.7"                                   # OpenShift version    
    export CLUSTER_IMAGE="quay.io/openshift-release-dev/ocp-release:4.7.21-x86_64"
    export CLUSTER_NAME="waiops"                                   # OpenShift cluster name    
    export CLUSTER_DOMAIN="redhat.local"                           # Domain name where my cluster will be deployed 
    export CLUSTER_NET_TYPE="OpenShiftSDN"                         # Set the Network type to deploy with OpenShift
    export CLUSTER_CIDR_NET="10.128.0.0/14"
    export CLUSTER_CIDR_SVC="172.30.0.0/16"
    export CLUSTER_HOST_NET="172.30.244.0/24"
    export CLUSTER_HOST_PFX="23"
    export CLUSTER_WORKER_HT="Enabled"
    export CLUSTER_WORKER_COUNT="0"
    export CLUSTER_MASTER_HT="Enabled"
    export CLUSTER_MASTER_COUNT="0"
    export API_VIP="192.168.2.103"                                  # Set the static IP address for API VIP
    export INGRESS_VIP="192.168.2.104"                              # Set the static IP address for Ingress VIP
    export MACHINE_CIDR_NET="192.168.2.0/24"                        # Set the Machine Network CIDR 
    #export HTTP_PROXY_URL="https:\\192.168.1.99:3129"              # Set the http proxy url http://<user>:password>@<ipaddr>:<port> 
    #export HTTPS_PROXY_URL="https:\\192.168.1.99:3129              # Set the https proxy url http://<user>:password>@<ipaddr>:<port> 
    #export NO_PROXY_DOMAINS="redhat.local"                         # Set the domains to bypass proxy        
    export CLUSTER_SSHKEY=$(cat ~/.ssh/id_rsa.pub)
    ```
    
5. Export the variables in your environment
    ```bash
    #source cluster-details.sh
    ```
    
6. Verify that you can communicate with API with following curl command
    ```bash
    curl -s -X GET "https://$ASSISTED_SERVICE_API/api/assisted-install/v1/clusters" \
    -H "accept: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    | jq -r
    ```
    
7. Now download the pull secret from [here](https://cloud.redhat.com/openshift/install/pull-secret) and put it in a file pull-scret.txt
    ![Pull Secret](https://github.com/rh-telco-tigers/Assisted-Installer-API/blob/main/images/pull-secret.png)
    
8. Create a variable with raw content of pull-secret.txt file.
    ```bash
    PULL_SECRET=$(cat pull-secret.txt | jq -R .)
    ```
    
9. Create an Assisted-Service deployment.json file
    ```bash
    cat << EOF > ./deployment.json
    {
      "kind": "Cluster",
      "name": "$CLUSTER_NAME",
      "openshift_version": "$CLUSTER_VERSION",
      "ocp_release_image": "$CLUSTER_IMAGE",
      "base_dns_domain": "$CLUSTER_DOMAIN",
      "network_type": "$CLUSTER_NET_TYPE",
      "hyperthreading": "all",
      "cluster_network_cidr": "$CLUSTER_CIDR_NET",
      "cluster_network_host_prefix": $CLUSTER_HOST_PFX,
      "service_network_cidr": "$CLUSTER_CIDR_SVC",
      "user_managed_networking": false,
      "vip_dhcp_allocation": false,
      "api_vip": "$API_VIP"
      "ingress_vip": "$INGRESS_VIP"
      "host_networks": "$CLUSTER_HOST_NET",
      "http_proxy": "$HTTP_PROXY_URL",
      "https_proxy": "$HTTPS_PROXY_URL",
      "no_proxy": "$NO_PROXY_DOMAINS",
      "ssh_public_key": "$CLUSTER_SSHKEY",
      "pull_secret": $PULL_SECRET
    }
    EOF
    ```
    
10. REFRESH TOKEN:  (This may need to be performed periodically)
      ```bash
      export TOKEN=$(curl \
      --silent \
      --data-urlencode "grant_type=refresh_token" \
      --data-urlencode "client_id=cloud-services" \
      --data-urlencode "refresh_token=${OFFLINE_ACCESS_TOKEN}" \
      https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token | \
      jq -r .access_token)
      ```

11. Create the cluster via Assisted-Servcice API this will generate a "cluster id" whcih will need to be exported for future use.
    ```bash
    export CLUSTER_ID=$( curl -s -X POST "https://$ASSISTED_SERVICE_API/api/assisted-install/v1/clusters" \
     -d @./deployment.json \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     | jq '.id' )

    export CLUSTER_ID=$( sed -e 's/^"//' -e 's/"$//' <<<"$CLUSTER_ID")
   
    echo $CLUSTER_ID
    e85fc7d5-f274-4359-acc5-48044fc67132
    ```
     
12. Review your changes by issuing following curl command
    ```bash
    curl -s -X GET \
    --header "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    "https://$ASSISTED_SERVICE_API/api/assisted-install/v1/clusters/$CLUSTER_ID/install-config" \
    | jq -r
    
    apiVersion: v1
    baseDomain: redhat.local
    networking:
      networkType: OpenshiftSDN
      clusterNetwork:
      - cidr: 10.128.0.0/14
        hostPrefix: 23
      machineNetwork:
      - cidr: ""
      serviceNetwork:
      - 172.31.0.0/16
    metadata:
      name: waiops
    compute:
    - hyperthreading: Enabled
      name: worker
      replicas: 0
    controlPlane:
      hyperthreading: Enabled
      name: master
      replicas: 0
    platform:
      baremetal:
        provisioningNetwork: Disabled
        apiVIP: ""
        ingressVIP: ""
        hosts: []
      vsphere: null
    fips: false
    pullSecret: 'Your-Pull-Secret'
    sshKey: 'Your-SSH_KEY'
    ```
    
13. Now you will see a cluster created in https://console.redhat.com/openshift/assisted-installer/clusters
    ![AI Console](https://github.com/rh-telco-tigers/Assisted-Installer-API/blob/main/images/ai-console.png)


14. Click on cluster name for details. Review and click on "Next"
    ![cluster details](https://github.com/rh-telco-tigers/Assisted-Installer-API/blob/main/images/cluster-details.png)


15. GENERATE THE NMSTATE YAML FILES:
    Create a yaml file for each node in the cluster 
    (master-0, master-1, master-2, worker-0, worker-1, worker-2)
    The master-0 file is shown below. Replicate this for the other nodes, changing the IP address.
```bash

cat << EOF > ~/master-0.yaml 
dns-resolver:
  config:
    server:
    - 192.168.1.210
interfaces:
- ipv4:
    address:
    - ip: 192.168.2.100
      prefix-length: 24
    dhcp: false
    enabled: true
  name: ens192
  state: up
  type: ethernet
routes:
  config:
  - destination: 0.0.0.0/0
    next-hop-address: 192.168.2.1
    next-hop-interface: ens192
    table-id: 254
EOF
```

16. GENERATE THE DISCOVERY ISO FILE:
```bash
DATA=$(mktemp)

jq -n --arg SSH_KEY "$NODE_SSH_KEY" \
--arg NMSTATE_YAML1 "$(cat ~/master-0.yaml)" --arg NMSTATE_YAML2 "$(cat ~/master-1.yaml)" \
--arg NMSTATE_YAML3 "$(cat ~/master-2.yaml)" --arg NMSTATE_YAML4 "$(cat ~/worker-0.yaml)" \
--arg NMSTATE_YAML5 "$(cat ~/worker-1.yaml)" --arg NMSTATE_YAML6 "$(cat ~/worker-2.yaml)" \
'{
  "ssh_public_key": $SSH_KEY,
  "image_type": "full-iso",
  "static_network_config": [
    {
      "network_yaml": $NMSTATE_YAML1,
      "mac_interface_map": [{"mac_address": "00:50:56:b9:02:7b", "logical_nic_name": "ens192"}]
    },
    {
      "network_yaml": $NMSTATE_YAML2,
      "mac_interface_map": [{"mac_address": "00:50:56:b9:ff:58", "logical_nic_name": "ens192"}]
    },
    {
      "network_yaml": $NMSTATE_YAML3,
      "mac_interface_map": [{"mac_address": "00:50:56:b9:72:7d", "logical_nic_name": "ens192"}]
     },
    {
      "network_yaml": $NMSTATE_YAML4,
      "mac_interface_map": [{"mac_address": "00:50:56:b9:d8:09", "logical_nic_name": "ens192"}]
    },
    {
      "network_yaml": $NMSTATE_YAML5,
      "mac_interface_map": [{"mac_address": "00:50:56:b9:1e:92", "logical_nic_name": "ens192"}]
    },
    {
      "network_yaml": $NMSTATE_YAML6,
      "mac_interface_map": [{"mac_address": "00:50:56:b9:be:33", "logical_nic_name": "ens192"}]
     }
  ]
}' > $DATA


curl -X POST "https://$ASSISTED_SERVICE_API/api/assisted-install/v1/clusters/$CLUSTER_ID/downloads/image" \
  -H "Content-Type: application/json"  -H "Authorization: Bearer $TOKEN" -d @$DATA
```

17. DOWNLOAD THE DISCOVERY ISO FILE:
   ```bash
   curl -L "http://$ASSISTED_SERVICE_API/api/assisted-install/v1/clusters/$CLUSTER_ID/downloads/image" \
   -o ~/discovery-image-$CLUSTER_NAME-master0.iso  -H "Authorization: Bearer $TOKEN"
  ```

18. RETRIEVE THE AWS S3 DOWNLOAD URL (OPTIONAL):
   ```bash
   curl -s -X GET "https://$ASSISTED_SERVICE_API/api/assisted-install/v1/clusters/$CLUSTER_ID" \
   -H "Authorization: Bearer $TOKEN"|jq .image_info
   
     "download_url": "https://s3.us-east-1.amazonaws.com/assisted-installer/discovery-image-....", 
     "expires_at": "2021-08-19T07:11:46.229Z"
   ```

19. Boot your VM/Baremetal with the Discovery ISO.


20. In Host discovery Tab, once all of your nodes appear in the list, click on Next.
    ![discovery host](https://github.com/rh-telco-tigers/Assisted-Installer-API/blob/main/images/discovery-iso.png)

21. In the Networking tab, review the settings for the Cluster network CIDR, host prefix, and Service network CIDR. 
    Verify that these have been set as specified previously from in cluster-details file.

    From this tab, also enter the static IP addresses for the API VIP and the Ingress VIP. 

    ![Service IP](https://github.com/rh-telco-tigers/Assisted-Installer-API/blob/main/images/service-ip.png)

22. Proceed to click on Next and Install the cluster
