## Architecture Diagram

```
                Cert-Manager Objects                        Nginx1 Objects

               ┌───────────────────────┐                    ┌─────────────────────────────────┐
Created CA     │ kind: Secret          │                    │                                 │
private key ──►│ name: echo-app-secret │◄─────────┐         │ kind: Ingress                   │
and cert       │ tls.key: **priv key** │          │         │ name: echo-ingress              │
               │ tls.crt: **cert**     │          │         │ tls:                            │
               └───────────────────────┘          │         │   - hosts:                      │
                                                  │         │     - echo.app                  │
               ┌──────────────────────────────┐   │    ┌────┼───secretName: echo-app-tls      │
               │                              │   │    │    │                                 │
               │ kind: ClusterIssuer          │   │    │    └─────────────────────────────────┘
           ┌───┤►name: echo-app-clusterissuer │   │    │
           │   │ secretName: echo-app-secret ─┼───┘    │
           │   │                              │        │
           │   └──────────────────────────────┘        │
           │                                           │
           │   ┌───────────────────────────────┐       │
           │   │                               │       │
           │   │ kind: Certificate             │       │
           │   │ name: echo-app-cert           │       │
           └───┼─issuerRef:                    │       │
               │   name: echo-app-clusterissuer│       │
               │   kind: ClusterIssuer         │       │
               │ dnsNames:                     │       │
               │   - echo.app                  │       │
           ┌───┼─secretName: echo-app-tls      │       │
           │   │                               │       │
           │   └──────────┬────────────────────┘       │
           │              │                            │
           │              │ will be created            │
           │              ▼ and managed automatically  │
           │   ┌───────────────────────────────┐       │
           │   │                               │       │
           │   │ kind: Secret                  │       │
           └───┤►name: echo-app-tls      ──────┼───────┘
               │                               │
               └───────────────────────────────┘
```

# Self-Signed Certificates in Minikube

## Requirements
- Install `base64`
  Windows
  ```
  choco install base64
  ```
  Linux
  ```
  apt-get install base64
  ```
  MacOS
  ```
  brew install base64
  ```
- Install `openssl`
  Windows
  ```
  choco install openssl
  ```
  Linux
  ```
  apt-get install openssl
  ```
  MacOS
  ```
  brew install openssl
  ```
- Install `minikube`
  (https://minikube.sigs.k8s.io/docs/start/)

## Steps

1. Create a Certificate Authority
    a. Create a CA private key
    ```
    openssl genrsa -out ca.key 4096
    ```
    b. Create a CA certificate
    ```
    openssl req -new -x509 -sha256 -days 365 -key ca.key -out ca.crt
    ```
    c. Import the CA certificate in the `trusted Root Ca store` of your clients

   Note :
   Every OS having different ways to apply the certificate as a trust certificate in their OS

3. Create and apply to minikube
   
   You can copy my yaml file or clone it to your local directory

   a. Install Cert Manager
      ```
      curl -LO https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml
      kubectl apply -f cert-manager.yaml
      ```
   b. Since you need to convert the certificate that you genereated previosly to base64.
      
      You use this command to convert it

      Convert the content of the key and crt to base64 oneline in Linux
      ```
      cat ca.crt | base64 -w 0
      cat ca.key | base64 -w 0
      ```
      if you're not using Linux you can use link below to convert it

      (https://base64.guru/converter/encode/image)

      Copy the converted code to tls.crt and tls.key in ca-secret.yaml
   
   c. Apply Certificate use kubectl
      ```
      kubectl apply -f openssl-issuer/ca-secret.yml
      kubectl apply -f openssl-issuer/issuer.yml
      kubectl apply -f openssl-issuer/certificate.yml
      kubectl describe certificate -n echo
      kubectl get secrets -n echo
      ```
      make sure certificate is ready
      ```
      ...
      Status:
        Conditions:
          Last Transition Time:  2024-01-20T01:47:26Z
          Message:               Certificate is up to date and has not expired
          Observed Generation:   1
          Reason:                Ready
          Status:                True
          Type:                  Ready
        Not After:               2024-04-19T01:47:26Z
        Not Before:              2024-01-20T01:47:26Z
        Renewal Time:            2024-03-20T01:47:26Z
        Revision:                1
      Events:
        Type    Reason     Age   From                                       Message
        ----    ------     ----  ----                                       -------
        Normal  Issuing    19m   cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
        Normal  Generated  19m   cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "echo-app-cert-brb2j"
        Normal  Requested  19m   cert-manager-certificates-request-manager  Created new CertificateRequest resource "echo-app-cert-1"
        Normal  Issuing    19m   cert-manager-certificates-issuing          The certificate has been successfully issued
      ```
      ```
      NAME           TYPE                DATA   AGE
      echo-app-tls   kubernetes.io/tls   3      115m
      ```
   d. Enable Ingress
      ```
      minikube addons enable ingress
      ```
   e. Apply Service, Ingress, and Deploy
      ```
      kubectl apply -f service/ingress-test-with-tls.yml
      kubectl get pods
      ```
      Check your pod is already running or not
      ```
      NAME                           READY   STATUS      RESTARTS        AGE
      echo-app                       1/1     Running     0               87m
      ```
      ```
      kubectl get ingress
    
      NAME           CLASS   HOSTS      ADDRESS          PORTS     AGE
      echo-ingress   nginx   echo.app   172.27.166.228   80, 443   108s
      ```
4. Add your dns to `/etc/hosts`
5. Try access the dns in your browser
6. In my case I run minikube with ```--driver=docker``` so I need to run a tunnel in different terminal
   ```minikube tunnel```
   docs detail at (https://base64.guru/converter/encode/image) at Ingress tab

