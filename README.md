# SealedSecret


Managing secrets in kubernetes is really challenging when we have to provide the secret value/string in a yaml file. 
Although kuberntes stores the secrets in encoded form but decoding this secret is not rocket science, we can easily decode the value and get the password/secret.

The actual problem starts when we have to share this secret file within the team or with community over version control system. 

Either we have to provide wrong password and create the secret or not provide the password at all. This approach is fine when we are shaing with a community but within the team and where the autoamtion is required, we have to provide the actual file.


To overcome from this situation, we can use `Sealed Secrets`

</br>


</br>

### POC Scope
---------------

1. To create a kubernetes sealed-secret using sealed-secrets-controller.
2. Decrypt this sealed-secret again to actaully create a kubernetes secret.
3. Attach the kubernetes secret with jenkins as Credentials which will be used by Jobs


</br>

### How sealed-secrets helps us
---------------------------------

1. We install the sealed-secret controller in our kubernetes cluster, by default in kube-system namespace. This controller has one public certificate and a private key.

2. The controller 's public certificate will be used to seal the kubernetes secret.

3. Once we seal (encrypt) the kubernetes secret using the controller public certificate, this will only be decrypted by this same kubernetes cluster 's controller private key.

4. Since no other key can decrypt this sealed secret, we can share it with anyone on any platform.


</br>

### Pre-Requisites
--------------------

1. A kubernetes cluster where we will install sealed-secret-controller.
2. Kubeseal package --> This is the CLI tool which we talk controller and encrypt our secret.


        For mac OS ---> brew install kubeseal

        For other distributions ---> https://github.com/bitnami-labs/sealed-secrets/releases


</br>

### Create the sealed-secrets controller
---------------------------------------

This repo contains one yaml file which has all the required components to create the sealed-secrets-controller.

        kubectl apply -f controller.yaml


Once the controller Pod is ready, you can see the certificate by checking the logs of the pod 

        kubectl logs <sealed-secrets-controller-pod> -n kube-system

if you want, you can keep this certificate in some file.   

- you can also get this certificate key by describing the sealed-secret secret created in kube-system namespace.

- you can save the certificate by running below command as well

        kubeseal --fetch-cert > cert.pem

- In case you created the controller in some other namespace, use below command to get the certificate

        kubeseal --controller-namespace kube-system --fetch-cert > cert.pem

</br>

### Creating kubernetes secret
--------------------------------

Now we can create one kubernetes secret for testing purpose. 

1. First base64 encode your password or string 

            echo -n 'secretpassword' | base64

2. Create the secret by putting the encoded string in k8s-secret.yaml file            

            kubectl apply -f k8s-secret.yaml        


3. At this moment your kubernetes secret is ready as usual which we were keeping in git so far.


</br>

### Seal (Encrypt) the k8s secret using sealed-secrets
----------------------------------------------------------

Now we are going to seal our secret using the sealed-secrets-controller and created a sealed-secret.yaml file.

        kubeseal < k8s-secret.yaml -o yaml > sealedsecret.yaml

        OR in case your controller is somewhere else and you have the cert file available,  

        kubeseal < k8s-secret.yaml --cert cert.pem -o yaml > sealedsecret.yaml
 

By default this sealed secret scope is strict which means you can decrypt/ create the actual secret from this new file only in the defined namespace (which we gave in our original k8s-secret.yaml file).


</br>


### Recreate the secret from the sealed-secrets file
-----------------------------------------------------


Now assume that the sealedsecret.yaml file was uploaded on the github and you have downloaded it to create the actual secret in your environment.

We can do it by simply running the below command-

        kubectl apply -f sealedsecret.yaml

NOTE- 

- This will create the actual secret in default namespace only if the encryption was done by this cluster 's controller certificate.

- If this sealed secret was created by some other cluster, you will not be able to create it.  



</br>

## Integrating kubernetes secrets with Jenkins
-------------------------------------------------

We can manage jenkins credentials directly on the jenkins console however, sometimes we want to manage these credentials from kubernetes secrets for greater control.

IMP: 
- `To make use of kubernetes secrets with Jenkins, Our Jenkins server must be running as a POD in the cluster.`
- `Also, the service account which is used to create Jenkins pod, must have secret list permissions.`


1. While creating the kubernetes secret to attach to Jenkins, we have to define some annotations along with our secret. A sample secret file is attached to this repo as 'k8s-secret-for-jenkins.yaml'. Note the username and password is base64 decoded.

            kubectl apply -f k8s-secret-for-jenkins.yaml

2. We need to install the required plugin in Jenkins server from manage plugin section. The plugin name is - "Kubernetes Credentials Provider"

3. Once done, you should be able to see the kubernetes secret as credentials on Jenkins Dashboard.


</br>


Reference-

- https://github.com/bitnami-labs/sealed-secrets
- https://plugins.jenkins.io/kubernetes-credentials-provider/#documentation
