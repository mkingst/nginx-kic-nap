# NGINX Plus KIC with NAP Pod 

This is an NGINX Plus + App Protect pod that runs in Kubernetes/OpenShift. It is not an ingress controller, and it does not take ingress resources. 

It is configured using the standard NGINX Configuration files, which are considered configmaps in Kubernetes.

We also have the option to use a custom WAF policy (See note at the end), but the default configuration uses the strict policy. 

# 1. Build the N+ and NAP Image

Build the NGINX Plus and App Protect Image by copying the following files into a directory:
 
Dockerfile                           
entrypoint.sh                    
nginx-repo.key                 
custom_log_format.json              
nginx-repo.crt                   
 nginx.conf
 
Note: the cert and key needs to be obtained from F5. 
 
Then run:
 
<code>docker build --no-cache -t nginx-plus-nap . </code>
 
Push this image to a local repository.

# 2. Deploy a standard NGINX Configuration  
 
Use the following config Map containing NGINX configuration
<code>01nginx-config-default.yaml</code>
 
<b>Note: Change the syslog servers ClusterIP </b>

# 3. Deploy the NGINX Pod  
 
Deployment YAML to Deploy NGINX Plus and App Protect pod:
<code>02nginx-plus-default.yaml</code>

# 4. Create a service for NGINX Plus (listening on port 80 by default)
 
<code>
kubectl create service clusterip nginx-plus-nap  --tcp=80:80
</code>
 
# 5. Configure your Ingress Controller to route to the NGINX Plus service on port 80 (can be changed).

Standard Ingress

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  tls:
  - hosts:
    - nginx.example.com
    secretName: nginx-secret
  rules:
  - host: nginx.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-plus-nap
          servicePort: 80
```

This is a VirtualServer CRD if you are using NGINX Plus:

```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: nginx-plus-nap
spec:
  host: nginx.example.com
  upstreams:
  - name: nginx-plus-nap
    service: nginx-plus-nap
    port: 80
  routes:
  - path: /
    action:
      pass: nginx-plus-nap
 ```


# Notes

Use 01nginx-config-custom.yaml and 02nginx-plus-custom.yaml if you want to use your own policy and log configuration. 

These assume that you have configmaps created for the WAF policy and log format:

```yaml
          volumeMounts:
          - name: nginx-config
            mountPath: /etc/nginx/conf.d/nginx-plus.conf
            subPath: nginx-plus.conf
          - name: my-policy
            mountPath: /etc/app_protect/conf/custom_policy.json
            subPath: custom_policy.json
          - name: default-logconf
            mountPath: /etc/app_protect/conf/default_logconf.json
            subPath: default_logconf.json
```

```yaml
      volumes:
        - name: nginx-config
          configMap:
            name: nginx-config
        - name: my-policy
          configMap:
            name: custom-policy
        - name: default-logconf
          configMap:
            name: default-logconf
```

The following are two examples:

<b>custom-policy.json: </b>

```json
   
   {"policy":{"applicationLanguage":"utf-8","blocking-settings":{"evasions":[{"description":"Apache whitespace","enabled":true},{"description":"Bad unescape","enabled":true},{"description":"Bare byte decoding","enabled":true},{"description":"Directory traversals","enabled":true},{"description":"IIS Unicode codepoints","enabled":true},{"description":"IIS backslashes","enabled":true},{"description":"Multiple decoding","enabled":true,"maxDecodingPasses":2}],"http-protocols":[{"description":"Header name with no header value","enabled":true}],"violations":[{"alarm":true,"block":false,"name":"VIOL_DATA_GUARD"},{"alarm":true,"block":true,"name":"VIOL_HTTP_PROTOCOL"},{"alarm":true,"block":true,"name":"VIOL_FILETYPE"},{"alarm":true,"block":true,"name":"VIOL_HEADER_METACHAR"},{"alarm":true,"block":true,"name":"VIOL_EVASION"}]},"data-guard":{"creditCardNumbers":true,"enabled":true,"enforcementMode":"ignore-urls-in-list","enforcementUrls":[],"maskData":true,"usSocialSecurityNumbers":true},"enforcementMode":"blocking","filetypes":[{"allowed":true,"checkPostDataLength":false,"checkQueryStringLength":true,"checkRequestLength":false,"checkUrlLength":true,"name":"*","postDataLength":4096,"queryStringLength":2048,"requestLength":8192,"responseCheck":false,"type":"wildcard","urlLength":2048},{"allowed":false,"name":"bat"}],"general":{"trustXff":true},"name":"nginx-policy","signature-sets":[{"alarm":true,"block":true,"name":"Command Execution Signatures"},{"alarm":true,"block":true,"name":"Cross Site Scripting Signatures"},{"alarm":true,"block":true,"name":"SQL Injection Signatures"}],"signature-settings":{"minimumAccuracyForAutoAddedSignatures":"low"},"template":{"name":"POLICY_TEMPLATE_NGINX_BASE"}}}

```

<code>kubectl create configmap custom-policy --from-file custom-policy.json</code>



<b>default_logconf.json: </b>


```json
    {"content":{"format":"default","max_message_size":"64k","max_request_size":"any"},"filter":{"request_type":"all"}}
```


<code>
kubectl create configmap default-logconf --from-file default_logconf.json
</code>

If you update a configmap, you need to redeploy the pod.

<code>kubectl rollout restart deployment nginx-plus-nap </code>


