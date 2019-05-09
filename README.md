### Secure your kubernetes cluster with nginx ingress with TLS and LetsEncrypt.

`Note` : Make sure you have intalled helm in your pc.

* Role : Create role for accessing helm to the cluster.
```
$ kubectl create clusterrolebinding tiller-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:default
```
```
$ helm init
```

* Nginx : Install Nginx ingress using helm.
```
$ helm install stable/nginx-ingress --namespace kube-system
```

* Deploy : Deploy Sample Example App.
We will deploy nginx webserver in our cluster and access it with nginx ingress, You can deploy whatever app you want.
```
$ helm install stable/nginx --name nginx-app
```


* Expose : Expose it to the Cluster IP.
Expose the deployed nginx app to the cluster ip so that ingress can communicate with it.
```
$ kubectl expose deployment nginx-app --type=ClusterIP
```



* Ingress : Create Ingress object to access.

Now we will create Nginx ingress to access our app.


```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myapp
  annotations:
    kubernetes.io/ingress.class: nginx
    # Add to generate certificates for this ingress
spec:
  rules:
    - host: imvishalvyas.ml
      http:
        paths:
          - backend:
              serviceName: myapp
              servicePort: 80
            path: /

```

Save the file and apply the ingress.

```
$ kubectl apply -f basic-ingress.yaml
```


After some moments you can access your site {deployment} from nginx ingress controller external. You can find that external ip from below command.
```
$ kubectl --namespace kube-system get services -o wide -w funky-labradoodle-nginx-ingress-controller
```




* Configure TLS with LetsEncrypt and Kube-Lego.
Now our app is ready and working over http but we want to make it secure, So we will configured TLS for our app, Kubelego will install letsenrypt cert for your app, LetsEncrypt is free TLS certificate authority, Run the below command to install and configure Kube-Lego chart using helm and make sure that you have changed your email address.

```
helm install stable/kube-lego --namespace kube-system --set config.LEGO_EMAIL=YOUR_EMAIL_ID,config.LEGO_URL=https://acme-v01.api.letsencrypt.org/directory
```

Now we have to Modify TLS settings in our ingress file with our domain name and apply it.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myapp
  annotations:
    kubernetes.io/ingress.class: nginx
    # Add to generate certificates for this ingress
    kubernetes.io/tls-acme: 'true'
spec:
  rules:
    - host: imvishalvyas.ml
      http:
        paths:
          - backend:
              serviceName: myapp
              servicePort: 80
            path: /
  tls:
    # With this configuration kube-lego will generate a secret in namespace foo called `example-tls`
    # for the URL `www.example.com`
    - hosts:
        - "imvishalvyas.ml"
      secretName: kube-lego-ssl

```



Save the ingress file and apply new changes.

```
$ kubectl apply -f tls-ingress.yaml
```


Now you can access your website using your domain name and with SSL.

```
curl https://mywebsite.com
```


# `Optional`

### Manually configure TLS.
```
$ kubectl create secret tls yourwebsite-ssl-secret --key /path/tls.key --cert /path/tls.crt
```
this command will create secret key name 'yourwebsite-ssl-secret' with the certificate. now we have to add them in ingress file like below.

```
  tls:

    # With this configuration kube-lego will generate a secret in namespace foo called `example-tls`

    # for the URL `www.example.com`

    - hosts:

        - "vishalvyas.com"

      secretName: mysite-tls
```

### Configure Multiple Domain Nginx Ingress Kubernetes

We can configure and manage multiple domain in single kubernetes cluster using nginx ingress. you need to just update your nginx file 'spec' like below. Also we can use path base routing in ingress. you can see in 1st host abc.com i have use multipath routing. We can access it from abc.com and also abc.com/

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myapp
  annotations:
    kubernetes.io/ingress.class: nginx
    # Add to generate certificates for this ingress
    kubernetes.io/tls-acme: 'true'
spec:
  rules:
    - host: imvishalvyas.ml
      http:
        paths:
          - backend:
              serviceName: myapp
              servicePort: 80
            path: /
    - host: linuxguru.com
    http:
      paths:
      - backend:
          serviceName: apache
          servicePort: 80
        path: /
  tls:
    # With this configuration kube-lego will generate a secret in namespace foo called `example-tls`
    # for the URL `www.example.com`
    - hosts:
        - "imvishalvyas.ml"
      secretName: imvishalvyas-tls
    - hosts:
        - "linuxguru.com"
      secretName: linuxguru-tls
```



* How to assign static ip to the nginx ingress.

Use the below command while installing nginx ingress controller kubernetes, you will have to define your static ip to the command, it will allocate your static ip to the ingress.

```
$ helm install stable/nginx-ingress --namespace kube-system --set controller.service.loadBalancerIP=myip --set rbac.create=true
```

`Note`: ingress controller and static ip should be in same region.
