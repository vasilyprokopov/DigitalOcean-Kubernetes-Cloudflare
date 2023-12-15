# Cloudflare with DigitalOcean Kubernetes for DDoS, WAF, and Other Services

## Problem Statement
DigitalOcean offers a managed Kubernetes service (DOKS) with basic DDoS protection. When running an application in DOKS, users might require advanced DDoS protection, a Web Application Firewall (WAF), and other similar services that Cloudflare provides. This tutorial will describe how to integrate Cloudflare services with DOKS.

## Architecture
Let's imagine a web application running in DOKS. This application is a Kubernetes Deployment with a Service that publicly exposes it via DigitalOcean's managed load balancer. To access the application, a client connects via HTTPS to Cloudflare. Cloudflare inspects the request. If the request is malicious, Cloudflare drops it. If the request is legitimate, it gets proxied to DigitalOcean over HTTPS. The load balancer on DigitalOcean receives the request, terminates TLS, and redirects it unencrypted over HTTP to one of the worker nodes. Terminating TLS on a load balancer is convenient for maintaining the TLS certificate at a single central point, thus avoiding the need to manage TLS certificates in a decentralized manner on worker nodes. The application then responds to the client through the same chain of proxies.

## Configuration Prerequisites
Ensure the following tools are configured on your computer:
- doctl
- kubectl
- jq

## Configuration

### Adding a Site to Cloudflare
Have your domain name registered and ready. Use it to add a new site in Cloudflare. Cloudflare will manage the DNS records for this domain. Make sure to update your authoritative DNS servers, or nameservers, at your domain registrar (e.g., GoDaddy, Namecheap) to point to Cloudflare.

### Enabling HTTPs End-to-End
In the Cloudflare control panel, navigate to your domain > SSL/TLS > Edge Certificates and enable "Always Use HTTPS". This ensures that traffic is always encrypted between the client and Cloudflare.

In the Cloudflare control panel, go to your domain > SSL/TLS > Overview and change the encryption mode to "Full (strict)". This ensures that traffic is encrypted not only between the client and Cloudflare but also between Cloudflare and DigitalOcean.

To encrypt traffic between Cloudflare and DigitalOcean, you will need to add a certificate to DigitalOcean's load balancer. This step will be covered later. For now, create and download this certificate from Cloudflare. Go to your domain > SSL/TLS > Origin Server and click "Create Certificate". Do not change the Key Format from PEM. Copy the data from the "Origin Certificate" and save it as `origin_certificate.pem`. Copy the data from the "Private Key" and save it as `private_key.pem`.

At this point, we have configured everything needed in Cloudflare. We can now proceed to DigitalOcean.

### Importing Origin Certificate to DigitalOcean
Now that you have two `.pem` files on your machine, you can import the origin certificate into DigitalOcean.

```bash
doctl compute certificate create --type custom --name cloudsandboxcert --leaf-certificate-path origin_certificate.pem --private-key-path private_key.pem
```

### Creating DOKS Cluster
Create a DOKS cluster on DigitalOcean. For example, I'm creating a cluster with two worker nodes in Frankfurt. Here's the doctl command:
```bash
doctl kubernetes cluster create my-cluster --count 2 --size s-1vcpu-2gb
```

To manage the cluster, you need to add an authentication token or certificate to your kubectl configuration file:
```bash
doctl kubernetes cluster kubeconfig save my-cluster
```

I'm running a sample application in Kubernetes with a Deployment of two Pods and a Service that exposes this application through a managed DigitalOcean load balancer on ports 80 and 443. Here's the manifest file.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - image: ealen/echo-server:latest
        name: echoserver
        ports:
        - containerPort: 80
        env:
        - name: PORT
          value: "80"

---
apiVersion: v1
kind: Service
metadata:
  name: hello-world-service
  annotations:
    service.beta.kubernetes.io/do-loadbalancer-protocol: "https"
    service.beta.kubernetes.io/do-loadbalancer-tls-ports: "443"
    service.beta.kubernetes.io/do-loadbalancer-redirect-http-to-https: "true"
    service.beta.kubernetes.io/do-loadbalancer-certificate-id: "a7888ee6-98b9-415d-bc05-487zzz8b"
    service.beta.kubernetes.io/do-loadbalancer-disable-lets-encrypt-dns-records: "true"
    service.beta.kubernetes.io/do-loadbalancer-allow-rules: "cidr:103.21.244.0/22,cidr:173.245.48.0/20,cidr:103.21.244.0/22,cidr:103.22.200.0/22,cidr:103.31.4.0/22,cidr:141.101.64.0/18,cidr:108.162.192.0/18,cidr:190.93.240.0/20,cidr:188.114.96.0/20,cidr:197.234.240.0/22,cidr:198.41.128.0/17,cidr:162.158.0.0/15,cidr:104.16.0.0/13,cidr:104.24.0.0/14,cidr:172.64.0.0/13,cidr:131.0.72.0/22"
spec:
  type: LoadBalancer
  selector:
    app: hello-world
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
    - name: https
      protocol: TCP
      port: 443
      targetPort: 80
```

Add your origin certificate ID to `service.beta.kubernetes.io/do-loadbalancer-certificate-id`.
This is the certificate we imported in a previous step. You can use doctl to list the available certificates:
```bash
doctl compute certificate list
```

Note that the allow rules in `service.beta.kubernetes.io/do-loadbalancer-allow-rules` only list Cloudflare's IP ranges. This means that the load balancer will only accept traffic coming from Cloudflare. Clients won't be able to circumvent Cloudflare by directly connecting to a load balancer's IP address on DigitalOcean. An up-to-date list of Cloudflare's IPs is available here: [Cloudflare IP Ranges](https://www.cloudflare.com/ips/)

You can create this application in Kubernetes by applying the manifest file:
```bash
kubectl apply -f manifest.yaml
```

After a few minutes, check the Kubernetes Services:
```bash
kubectl get services -o wide
```

Note down the EXTERNAL-IP address. In Cloudflare, navigate back to your domain > DNS > Records and add an A-record pointing to this address.

## Verification
- Traffic will get encrypted between client and Cloudflare.
- Traffic will get encrypted between Cloudflare and DigitalOcean load balancer.
- Traffic will not get encrypted between load balancer and Kubernetes worker nodes.
