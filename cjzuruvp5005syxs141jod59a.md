## Adding Domain Name and SSL Certificates for GoCD on Kubernetes Using Helm

_Recently, a  [colleague](https://www.linkedin.com/in/saranrajsekar/)  and I had to spike on the possibility of migrating our CI/CD infrastructure from a VM based design to a Kubernetes based one._

_While exploring how to do it, we had to spend some time to find out all the information we needed just to install and configure GoCD without the migration. So I decided to write this and a few other posts hoping someone like me would find it helpful. You can find other posts related to GoCD on GKE in my blog. _

GoCD's helm chart allows us to easily add a domain name and SSL certificate to it. This makes your connection to your Go Server more secure as it drastically reduces the chances of eaves-dropping and man-in-the-middle attacks. Here's how to do it

## Add an A record

First you need to add the Go server's external IP to an  [A record](https://my.bluehost.com/hosting/help/whats-an-a-record)  using your DNS provider. To get your Go server's IP in Kubernetes, you can use the following code

```
kubectl get ingress --namespace gocd gocd-server \
-o jsonpath="{.status.loadBalancer.ingress[0].ip}"

```
You can then add this IP in the DNS A record to a domain that you own to make the domain point to the IP.

## Add Domain to GoCD 

If you take a look at the default [values.yaml ](https://github.com/helm/charts/blob/master/stable/gocd/values.yaml), you can find the *`server.ingress`* section.

Now this section has a few fields as you can see below

```
ingress:
    # server.ingress.enabled is the toggle to enable/disable GoCD Server Ingress
    enabled: true
    # server.ingress.hosts is used to create an Ingress record.
    hosts:
      - cicd.mycustomdomain.com
```

In order to add a domain to your GoCD server you can simply add it under the hosts section. This will add the domain to the ingress as an allowed host. Please note that once this is done, the Go server cannot be accessed externally by using the Go server's IP address or by any other means except for the domain that you have specified.

That's it. Your Go Server dashboard can now be accessed by the domain that you have specified. However the connection to your Go Server is not secure. Not Yet!

## Add SSL Certificate to GoCD

In order to make your connection to the Go Sever secure, you first need to get a SSL certificate for your domain. There are multiple providers out there that can give your a certificate. Here's a helpful  [link](https://www.sslshopper.com/how-to-order-an-ssl-certificate.html). If you work in a corporate organization, you can probably ask your IT team for help.

Once you have a certificate, you have to apply the certificate to your ingress. You can use the same *`server.ingress`* section to apply that too. But first you have to create a Kubernetes `secret` that has the certificate and key. You can do this with the following command

```
kubectl create secret tls tls-secret \
--cert certificate.crt --key private_key.key
```
`certificate.crt` is the certificate that you have while `private_key.key` is the key that was used to generate the  [CSR](https://www.sslshopper.com/what-is-a-csr-certificate-signing-request.html) .

After creating the secret all you have to do is simply add the secret using the *`server.ingress`* section like this

``` 
  ingress:
    tls:
      - secretName: tls-secret
        hosts:
          - cicd.mycustomdomain.com
```

And that's it. You can now securely connect to your GoCD dashboard using your custom domain.

So there you go. I hope you find it useful. If you have any questions, please feel free to comment and I’ll answer from whatever I’ve learnt so far.

