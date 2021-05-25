# NGINX Plus KIC with NAP and DVWA

# Deploy DVWA App

<code>kubectl create deployment dvwa --image vulnerables/web-dvwa -o yaml --dry-run</code>

<code>kubectl create service clusterip dvwa  --tcp=80:80 -o yaml --dry-run </code>

# 2. Build the N+ and NAP Image and install

See here: https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/

Push this image to a local repository.

# 2. Deploy a standard NGINX Configuration  
 
Use the following config Map containing NGINX configuration
<code>01nginx-config-default.yaml</code>
 
<b>Note: Change the syslog servers ClusterIP </b>


# 3. Create a service for NGINX KIC (listening on port 80 by default)
 
<code>
kubectl create service nodeport nginx-ingress  --tcp=80:80,443:443
</code>
 
# 4. Deploy a syslog service

```kubectl apply -f 00syslog-setup.yaml ```

# 5. Deploy App Protect Log Format, an AP Policy, Signatures, and a Kubernetes Policy

```
kubectl apply -f 01ap-logconf.yaml;
kubectl apply -f 02dvwa-appolicy.yaml;
kubectl apply -f 03app-user-sig.yaml;
kubectl apply -f 04waf-policy.yaml;
```

# 6. Deploy a VirtualServer for DVWA and reference this policy

```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: dvwa-vs
spec:
  host: dvwa.example.com
  policies:
  - name: waf-policy
  upstreams:
  - name: dvwa 
    service: dvwa
    port: 80
  routes:
  - path: /
    action:
      pass: dvwa
```



# ELK Stack

This consists of Elastic Search, Logstash and Kibana.

1. Use the ELK yaml attached. 
2. Change the location of the volume so that it points to the 30-waf-logs-full-logstash.conf, which can be on your host. You can also use a configmap.
3. Configure NGINX App Protect (04waf-policy.yaml) to log to the clusterIP of the logstash service. 
4. Access Kibana on NodePort. 


