---
title: Ingress Controller (Traefik)
permalink: /docs/traefik/
description: How to configure Ingress Contoller based on Traefik in our Raspberry Pi Kuberentes cluster.
last_modified_at: "10-09-2022"
---

All HTTP/HTTPS traffic comming to K3S exposed services should be handled by a Ingress Controller.
K3S default installation comes with Traefik HTTP reverse proxy which is a Kuberentes compliant Ingress Controller.

Traefik is a modern HTTP reverse proxy and load balancer made to deploy microservices with ease. It simplifies networking complexity while designing, deploying, and running applications.

{{site.data.alerts.note}}

Traefik K3S add-on is disabled during K3s installation, so it can be installed manually to have full control over the version and its initial configuration.

K3s provides a mechanism to customize traefik chart once the installation is over, but some parameters like namespace to be used cannot be modified. By default it is installed in `kube-system` namespace. Specifying a specific namespace `traefik-system` for all resources that need to be created for configuring Traefik will keep kubernetes configuration cleaner than deploying everything on `kube-system` namespace.

{{site.data.alerts.end}}


## Traefik Installation

Installation using `Helm` (Release 3):

- Step 1: Add Traefik's Helm repository:

    ```shell
    helm repo add traefik https://helm.traefik.io/traefik
    ```
- Step2: Fetch the latest charts from the repository:

    ```shell
    helm repo update
    ```
- Step 3: Create namespace

    ```shell
    kubectl create namespace traefik-system
    ```
- Step 4: Create helm values file `traefik-values.yml`

  ```yml
  # Enabling prometheus metrics and access logs
  additionalArguments:
    - "--metrics.prometheus=true"
    - "--accesslog"
    - "--accesslog.format=json"
    - "--accesslog.filepath=/data/access.log"
  deployment:
    # Adding access logs sidecar
    additionalContainers:
      - name: stream-accesslog
        image: busybox
        args:
        - /bin/sh
        - -c
        - tail -n+1 -F /data/access.log
        imagePullPolicy: Always
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /data
          name: data
  service:
    spec:
      # Set load balancer external IP
      loadBalancerIP: 10.0.0.100
  # Enable cross namespace references
  providers:
    kubernetesCRD:
      enabled: true
      allowCrossNamespace: true  
  ```

- Step 5: Install Traefik

    ```shell
    helm -f traefik-values.yml install traefik traefik/traefik --namespace traefik-system
    ```

- Step 6: Confirm that the deployment succeeded, run:

    ```shell
    kubectl -n traefik-system get pod
    ```

### Helm chart configuration details


#### Enabling Prometheus metrics

By default helm installation does not enable Traefik's metrics for Prometheus. T

To enable that the following configuration must be provided to Helm chart:

```yml
additionalArguments:
  - "--metrics.prometheus=true"
```
This configuration makes traefik pod to open its metric port at TCP port 9100.


#### Assign a static IP address from LoadBalancer pool to Ingress service

Traefik service of type LoadBalancer created by Helm Chart does not specify any static external IP address. To assign a static IP address belonging to Metal LB pool, helm chart parameters shoud be specified:

```yml
service:
  spec:
    loadBalancerIP: 10.0.0.100
```

With this configuration ip 10.0.0.100 is assigned to Traefik proxy and so, for all services exposed by the cluster.

#### Enabling Access log

Traefik access logs contains detailed information about every request it handles. By default, these logs are not enabled. When they are enabled (throug parameter `--accesslog`), Traefik writes the logs to `stdout` by default, mixing the access logs with Traefik-generated application logs.

To avoid this, the access log default configuration must be changed to write logs to a specific file `/data/access.log` (`--accesslog.filepath`), adding to traekik deployment a sidecar container to tail on the access.log file. This container will print access.log to `stdout` but not missing it with the rest of logs.

Default access format need to be changed as well to use JSON format (`--accesslog.format=json`). That way those logs will be parsed by Fluentbit and log JSON payload automatically decoded extracting all fields from the log. See Fluentbit's Kubernetes Filter `MergeLog` configuration option in the [documentation](https://docs.fluentbit.io/manual/pipeline/filters/kubernetes).

Following Traefik helm chart values need to be provided:

```yml
additionalArguments:
  - "--accesslog"
  - "--accesslog.format=json"
  - "--accesslog.filepath=/data/access.log"
deployment:
  additionalContainers:
    - name: stream-accesslog
      image: busybox
      args:
      - /bin/sh
      - -c
      - tail -n+1 -F /data/access.log
      imagePullPolicy: Always
      resources: {}
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      volumeMounts:
      - mountPath: /data
        name: data
```

This configuration enables Traefik access log writing to `/data/acess.log` file in JSON format. It creates also the sidecar container `stream-access-log` tailing the log file.


#### Enabling cross-namespaces references in IngressRoute resources

As alternative to standard `Ingress` kuberentes resources, Traefik's specific CRD, `IngressRoute` can be used to define access to cluster services. This CRD allows advanced routing configurations not possible to do with `Ingress` available Traefik's annotations.

`IngressRoute` resources only can reference other Traefik's resources, i.e: `Middleware` located in the same namespace.
To change this, and allow IngresRoute access resources defined in other namespaces, [`allowCrossNamespace`](https://doc.traefik.io/traefik/providers/kubernetes-crd/#allowcrossnamespace) Traefik helm chart value must be set to true.


The following values need to be specified within helm chart configuration.

```yml
# Enable cross namespace references
providers:
  kubernetesCRD:
    enabled: true
    allowCrossNamespace: true 
```

### Creating Traefik-metric Service

A Kuberentes Service must be created for enabling the access to Prometheus metrics

- Create Manfifest file for the dashboard service
  
  ```yml
  apiVersion: v1
  kind: Service
  metadata:
    name: traefik-metrics
    namespace: traefik-system
    labels:
      app.kubernetes.io/instance: traefik
      app.kubernetes.io/name: traefik
      app.kubernetes.io/component: traefik-metrics
  spec:
    type: ClusterIP
    ports:
      - name: metrics
        port: 9100
        targetPort: metrics
        protocol: TCP
    selector:
      app.kubernetes.io/instance: traefik
      app.kubernetes.io/name: traefik
  ```
- Apply manifest files

- Check metrics end-point is available

  ```shell
  curl http://<traefik-dashboard-service>:9100/metrics
  ```

### Enabling access to Traefik-Dashboard

A Kuberentes Service must be created for enabling the access to UI Dashboard

- Create Manfifest file for the dashboard service
  
  ```yml
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: traefik-dashboard
    namespace: traefik-system
    labels:
      app.kubernetes.io/instance: traefik
      app.kubernetes.io/name: traefik
      app.kubernetes.io/component: traefik-dashboard
  spec:
    type: ClusterIP
    ports:
      - name: traefik
        port: 9000
        targetPort: traefik
        protocol: TCP
    selector:
      app.kubernetes.io/instance: traefik
      app.kubernetes.io/name: traefik
  ```

- Create Ingress rules for accesing through HTTPS dashboard UI, using certificates automatically created by certmanager and providing a basic authentication mechanism.
  
  ```yml
  ---
  # HTTPS Ingress
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: traefik-ingress
    namespace: traefik-system
    annotations:
      # HTTPS as entry point
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
      # Enable TLS
      traefik.ingress.kubernetes.io/router.tls: "true"
      # Use Basic Auth Midleware configured
      traefik.ingress.kubernetes.io/router.middlewares: traefik-system-basic-auth@kubernetescrd
      # Enable cert-manager to create automatically the SSL certificate and store in Secret
      cert-manager.io/cluster-issuer: ca-issuer
      cert-manager.io/common-name: traefik.picluster.ricsanfre.com
  spec:
    tls:
      - hosts:
          - traefik.picluster.ricsanfre.com
        secretName: traefik-tls
    rules:
      - host: traefik.picluster.ricsanfre.com
        http:
          paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: traefik-dashboard
                  port:
                    number: 9000
  ---
  # http ingress for http->https redirection
  kind: Ingress
  apiVersion: networking.k8s.io/v1
  metadata:
    name: traefik-redirect
    namespace: traefik-system
    annotations:
      # Use redirect Midleware configured
      traefik.ingress.kubernetes.io/router.middlewares: traefik-system-redirect@kubernetescrd
      # HTTP as entrypoint
      traefik.ingress.kubernetes.io/router.entrypoints: web
  spec:
    rules:
      - host: traefik.picluster.ricsanfre.com
        http:
          paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: traefik-dashboard
                  port:
                    number: 9000
  ```

  See section below to learn details about how to configure Traefik to access cluster services.

- Apply manifests files

- Acces UI through configured dns: `https://traefik.picluster.ricsanfre.com/dashboard/`


## Configuring access to cluster services with Traefik

Standard kuberentes resource, `Ingress`, or specific Traefik resource, `IngressRoute` can be used to configure the access to cluster services through HTTP proxy capabilities provide by Traefik.

Following instructions details how to configure access to cluster service using standard `Ingress` resources where Traefik configuration is specified using annotations.


### Enabling HTTPS and TLS 

All externally exposed frontends deployed on the Kubernetes cluster should be accessed using secure and encrypted communications, using HTTPS protocol and TLS certificates. If possible those TLS certificates should be valid public certificates.

#### Enabling TLS in Ingress resources

As stated in [Kubernetes documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls), Ingress access can be secured using TLS by specifying a `Secret` that contains a TLS private key and certificate. The Ingress resource only supports a single TLS port, 443, and assumes TLS termination at the ingress point (traffic to the Service and its Pods is in plaintext).

[Traefik documentation](https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress/), defines several Ingress resource annotations that can be used to tune the behavioir of Traefik when implementing a Ingress rule.

Traefik can be used to terminate SSL connections, serving internal not secure services by using the following annotations:
- `traefik.ingress.kubernetes.io/router.tls: "true"` makes Traefik to end TLS connections
- `traefik.ingress.kubernetes.io/router.entrypoints: websecure` 

With these annotations, Traefik will ignore HTTP (non TLS) requests. Traefik will terminate the SSL connections. Depending on protocol (HTTP or HTTPS) used by the backend service, Traefik will send decrypted data to an HTTP pod service or encrypted with SSL using the SSL certificate exposed by the service.

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myingress
  annotations:
    # HTTPS entrypoint enabled
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    # TLS enabled
    traefik.ingress.kubernetes.io/router.tls: true
spec:
  tls:
  - hosts:
    - whoami
    secretName: whoami-tls # SSL certificate store in Kubernetes secret
  rules:
    - host: whoami
      http:
        paths:
          - path: /bar
            pathType: Exact
            backend:
              service:
                name:  whoami
                port:
                  number: 80
          - path: /foo
            pathType: Exact
            backend:
              service:
                name:  whoami
                port:
                  number: 80
```

SSL certificates can be created manually and stored in Kubernetes `Secrets`.

```yml
apiVersion: v1
kind: Secret
metadata:
  name: whoami-tls
data:
  tls.crt: base64 encoded crt
  tls.key: base64 encoded key
type: kubernetes.io/tls
```

This manual step can be avoided using Cert-manager and annotating the Ingress resource: `cert-manager.io/cluster-issuer: <issuer_name>`. See further details in [SSL certification management documentation](/docs/certmanager/).

#### Redirecting HTTP traffic to HTTPS

Middlewares are a means of tweaking the requests before they are sent to the service (or before the answer from the services are sent to the clients)
Traefik's [HTTP redirect scheme Middleware](https://doc.traefik.io/traefik/middlewares/http/redirectscheme/) can be used for redirecting HTTP traffic to HTTPS.

```yml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect
  namespace: traefik-system
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

This middleware can be inserted into a Ingress resource using HTTP entrypoint

Ingress resource annotation `traefik.ingress.kubernetes.io/router.entrypoints: web` indicates the use of HTTP as entrypoint and `traefik.ingress.kubernetes.io/router.middlewares:<middleware_namespace>-<middleware_name>@kuberentescrd` indicates to use a middleware when routing the requests.


```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myingress
  annotations:
    # HTTP entrypoint enabled
    traefik.ingress.kubernetes.io/router.entrypoints: web
    # Use HTTP to HTTPS redirect middleware
    traefik.ingress.kubernetes.io/router.middlewares: traefik-system-redirect@kubernetescrd
spec:
  rules:
    - host: whoami
      http:
        paths:
          - path: /bar
            pathType: Exact
            backend:
              service:
                name:  whoami
                port:
                  number: 80
          - path: /foo
            pathType: Exact
            backend:
              service:
                name:  whoami
                port:
                  number: 80
```

### Providing HTTP basic authentication

In case that the backend does not provide authentication/autherization functionality (i.e: longhorn ui), Traefik can be configured to provide HTTP authentication mechanism (basic authentication, digest and forward authentication).

Traefik's [Basic Auth Middleware](https://doc.traefik.io/traefik/middlewares/http/basicauth/) can be used for providing basic auth HTTP authentication.

#### Configuring Secret for basic Authentication

Kubernetes Secret resource need to be configured using manifest file like the following:

```yml
# Note: in a kubernetes secret the string (e.g. generated by htpasswd) must be base64-encoded first.
# To create an encoded user:password pair, the following command can be used:
# htpasswd -nb user password | base64
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth-secret
  namespace: traefik-system
data:
  users: |2
    <base64 encoded username:password pair>
```

`data` field within the Secret resouce contains just a field `users`, which is an array of authorized users. Each user must be declared using the `name:hashed-password` format. Additionally all data included in Secret resource must be base64 encoded.

For more details see [Traefik documentation](https://doc.traefik.io/traefik/middlewares/http/basicauth/).

User:hashed-passwords pairs can be generated with `htpasswd` utility. The command to execute is:

```shell
htpasswd -nb <user> <passwd> | base64
```

The result encoded string is the one that should be included in `users` field.

`htpasswd` utility is part of `apache2-utils` package. In order to execute the command it can be installed with the command: `sudo apt install apache2-utils`

As an alternative, docker image can be used and the command to generate the `user:hashed-password` pairs is:
      
```shell
docker run --rm -it --entrypoint /usr/local/apache2/bin/htpasswd httpd:alpine -nb user password | base64
```
For example user:pass pair (oss/s1cret0) will generate a Secret file:

```yml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth-secret
  namespace: traefik-system
data:
  users: |2
    b3NzOiRhcHIxJDNlZTVURy83JFpmY1NRQlV6SFpIMFZTak9NZGJ5UDANCg0K
```
#### Middleware configuration

A Traefik Middleware resource must be configured referencing the Secret resource previously created

```yml
# Basic-auth middleware
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
  namespace: traefik-system
spec:
  basicAuth:
    secret: basic-auth-secret
```

#### Configuring Ingress resource

Add middleware annotation to Ingress resource referencing the basic auth middleware.
Annotation `traefik.ingress.kubernetes.io/router.middlewares:<middleware_namespace>-<middleware_name>@kuberentescrd` indicates to use a middleware when routing the requests.

```yml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami
  namespace: whoami
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: traefik-system-basic-auth@kubernetescrd
spec:
  rules:
    - host: whoami
      http:
        paths:
          - path: /bar
            pathType: Exact
            backend:
              service:
                name:  whoami
                port:
                  number: 80
          - path: /foo
            pathType: Exact
            backend:
              service:
                name:  whoami
                port:
                  number: 80

```
