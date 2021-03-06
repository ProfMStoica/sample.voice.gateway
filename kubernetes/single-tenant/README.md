# Deploying IBM Voice Gateway on Bluemix Kubernetes cluster or Kubernetes on a VM or on IBM Cloud private
Running the IBM Voice Gateway in a Kubernetes environment requires special considerations beyond the deployment of a typical HTTP based application. Because the voice gateway relies on SIP for call signaling and RTP for media which require affinity to a specific voice gateway instance, the Kubernetes ingress router needs to be avoided for these protocols. To work around the limitations of the ingress router, the voice gateway containers must be configuered in net Host mode, which means that when a port is opened in either of the voice gateway containers, those identical ports are also opened and mapped on the base VM. This also eliminates the need to define media port ranges in the kubectl configuration file which is not currently supported.

In Kubernetes terminology, a single voice gateway instance equates to a single Pod which contains both a SIP Orchestrator container and a Media Relay container. Only one POD should be deployed per node and the POD should be deploy with hostNetwork set to true. This will ensure that the SIP and media ports would be opened on the host VM and visible by the SIP Load Balancer.  

## Script defaults:

* Enforces one POD per node. If replicas (PODs > #nodes, the extra replica to be scheduled will remain in a waiting state)
* Exposes SIP and media relay ports on the associated VM by setting hostNetwork to true
* Auto restart of any failed containers

# Deploying Voice Gateway in single-tenant mode:

1) Edit the deploy.yaml file and add your WATSON_CONVERSATION_WORKSPACE_ID.

1) Create a secret for the Watson service APIKEYs using the following command (make sure to use your own APIKEY):
   ```bash
   kubectl create secret generic secret-creds \
   --from-literal=WATSON_STT_APIKEY='aaaBBBcc2hskx44mdcdd_Ind3' \
   --from-literal=WATSON_TTS_APIKEY='aaaBBBcc2hskx44mdcdd_Ind3' \
   --from-literal=WATSON_CONVERSATION_APIKEY='aaaBBBcc2hskx44mdcdd_Ind3'
   ```

1) If you want to enable recording (Optional): 
   - Set ENABLE_RECORDING to true in deploy.yaml and create the recording [PersistentVolume and PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) using the recording-pv.yaml and recording-pvc.yaml templates.
   - Update the recording template files  and run the following commands: 
     ```bash
     kubectl create -f recording-pv.yaml
     kubectl create -f recording-pvc.yaml
     ```
   - Uncomment recording volume and volumeMounts sections of the deploy.yaml

1) If you want to use MRCPv2 config file (Optional):
   - More info: [Configuring services with MRCPv2](https://www.ibm.com/support/knowledgecenter/SS4U29/MRCP.html)
   - Create unimrcpConfig secret from the unimrcpclient.xml file using the following command: 
    ```bash
    kubectl create secret generic unimrcp-config-secret --from-file=unimrcpConfig=unimrcpclient.xml
    ```
   - Uncomment the unimrcpconfig volume and volumeMounts sections of the deploy.yaml 
  
1) Deploy Voice Gateway:  
   ```bash
   kubectl create -f deploy.yaml
   ```

# SSL configuration
- More info: [Configuring SSL and TLS encryption](https://www.ibm.com/support/knowledgecenter/SS4U29/security.html#configuring-ssl-and-tls-encryption)

##### Adding trusted certificates for the SIP Orchestrator (For enabling SSL or Mutual Authentication):
- Create secret from the trust store key file:
  ```
  kubectl create secret generic trust-store-file-secret --from-file=trustStoreFile=myPKCS12File.p12 -n <namespace>
  ```
- Create secret for the SSL Passphrase:
  - Add passphrase in a text file `ssl_passphrase.txt` (Make sure there are no extra spaces or new lines in the text file)
  - Create secret from the text file:
    ```
    kubectl create secret generic ssl-passphrase-secret --from-file=sslPassphrase=ssl_passphrase.txt -n <namespace>
    ```
- Uncomment *ssl-so* in `volumes` and `volumeMounts` section of the deploy.yaml
- Uncomment variables `SSL_KEY_TRUST_STORE_FILE`, `SSL_FILE_TYPE` and `SSL_PASSPHRASE` in *deploy.yaml* and set `SSL_FILE_TYPE` to the correct file type.

##### Adding trusted certificates for the Media Relay (For enabling SSL):
- Create secret from client CA certificate file:
  ```
  kubectl create secret generic client-ca-cert-secret --from-file=clientCaCertFile=ca-bundle.pem -n <namespace>
  ```
- Uncomment *ssl-mr* in `volumes` and `volumeMounts` section of the deploy.yaml
- Uncomment variable `SSL_CLIENT_CA_CERTIFICATE_FILE` in *deploy.yaml* 

##### Adding certificates for the Media Relay (For Mutual Authentication):
- Create secret from the SSL client PKCS12 file:
  ```
  kubectl create secret generic ssl-client-pkcs12-file-secret --from-file=clientPkcs12File=myPKCS12File.p12 -n <namespace>
  ```
- Create secret for the SSL Passphrase:
  - Add passphrase in a text file `ssl_client_passphrase.txt` (Make sure there are no extra spaces or new lines in the text file)
  - Create secret from the text file:
    ```
    kubectl create secret generic ssl-client-passphrase-secret --from-file=sslClientPassphrase=ssl_client_passphrase.txt -n <namespace>
    ```
- Uncomment *ssl-mr-mutualauth* in `volumes` and `volumeMounts` section of the deploy.yaml
- Uncomment variables `SSL_CLIENT_PKCS12_FILE` and `SSL_CLIENT_PASSPHRASE` in *deploy.yaml*
