---
layout: post
title: Kubernetes Reverse Proxy Deployment
tags: [python, Automation, Kubernetes,Docker,Helm ]
image: /img/posts/201910/kubernetes.png
bigimg: /img/posts/201910/health-banner.png
---

I wanted to share my experience with this project where I created a Kubernetes Cluster with 2 apps that talk to each other.
Goal: Create an application that will be fronted by a Proxy Server that will act as a load balancer for the app servers. 

I'll be  creating the infrastructure on Kubernetes. This means that we will have 2 types of services, 1 for the proxy and another for app servers. Then I'll be creating 2 services of the type `app-server` and each one will have 3 replicas (meaning 3 servers) to ensure high availability. This service will be self-healing in the sense that if a replica goes down it will automatically be recreated by Kubernetes.

TODO: I have not completed everything I would have liked for this initial part of the project but I wanted to mention some of the enhancements I will add:
* Like I stated before, the infrastructure is self-healing but I would add a `liveness Probe` to the deployment, to monitor not only the health of the container but also monitor the health of the application by probing for HTTP errors that would indicate the state of the app.
* I would enhance the proxy application by adding a fail-over mechanism that would prevent errors from reaching end-users. This would be achieved by sending a request to an alternate service in the event of a HTTP 5xx server error response from the application server. Also add horizontal scaling based on the load of the server.


## Project Components:
* Application Development/Implementation
  * Python APP Server
  * Python Proxy Server
* Deployment/Automation
  * Kubernetes
  * helm
* Testing
 


# Application Development
## APP Server

[This application](https://github.com/roymartinezblanco/Kubernetes-Reverse-Proxy-Deployment/blob/master/app/app-server.py) will respond to HTTP request with a JSON body with a 200 ok message. It will also listen for the HTTP requests with the path `/fail` which will cause the application server to die.

Functions/Classes:
* startServer: Creates the httpServer and configures it to listen on TCP Port 8000
```python
def startServer():
    port = 8000
    ip = '0.0.0.0'
    configure_error_logging()
    logger.info('http app server is running')

    httpd = HTTPServer((ip,port), HTTPRequestHandler)
    httpd.serve_forever()
```
* configure_error_logging: Uses the logging liberty and logs all events to /tmp/proxy/event.log 
```python
def configure_error_logging():
    logger.setLevel(logging.DEBUG)
    # Format for our loglines
    formatter = logging.Formatter("[%(asctime)s] - %(name)s - %(levelname)s - %(message)s")
    # Setup console logging
    ch = logging.StreamHandler()
    ch.setLevel(logging.DEBUG)
    ch.setFormatter(formatter)
    logger.addHandler(ch)
    directory = "/tmp/proxy/"
    if not os.path.exists(directory):
        os.makedirs(directory)
    LOG_FILENAME = directory+"event.log"
    print(LOG_FILENAME)
    # Setup file logging as well
    fh = logging.FileHandler(LOG_FILENAME)
    fh.setLevel(logging.DEBUG)
    fh.setFormatter(formatter)
    logger.addHandler(fh)
```
* HTTPRequestHandler/do_GET: Handle for HTTP Get Requests. It creates a HTTP response body and simulates an outage.
```python
class HTTPRequestHandler(BaseHTTPRequestHandler):
    protocol_version = 'HTTP/1.1'
    def do_GET(self, body=True):
            try:
                #Recreate Server Header to obscure Software Versions
                self.server= ""
                self.server_version = "APP Server"
                self.sys_version = ""

                if self.path == "/fail":
                    sys.exit()
                else:
                    self.send_response_only(200)
                    body = {"message":"you got this!"}
                # Configure HTTP Response Headers
                self.send_header('Server','Jeju')
                x = datetime.datetime.now()
                self.send_header('Date',x.strftime("%c"))
                self.send_header('Content-type','application/json')
                body = {"message":"you got this!"}
                self.send_header('Content-Length',len(json.dumps(body).encode()))
                self.end_headers()
                # END Headers
                # Send Body

                self.wfile.write(json.dumps(body).encode())
                
                self.wfile.flush() #actually send the response if not already done.
                self.close_connection= 1 #Close Connection
                # Log Request
                logger.debug("{} - {} - {} ".format(self.address_string(),self.requestline,200))
            except Exception as e:
                # Log Error
                self.log_error("Request got ab error out: %r", e)
                logger.error("{} - {} - {} ".format(self.address_string(),self.requestline,200))
                self.close_connection = 1
                return
```


### Deploy APP Server Docker Image
Because we need to deploy this application to Kubernetes we need to create an image that we will later deploy in our cluster. 

To do this I created my image with this [dockerfile](https://github.com/roymartinezblanco/Kubernetes-Reverse-Proxy-Deployment/blob/master/docker/app-server.py) (All Docker files can be found under the /docker/ directory.). I chose to use alpine since its lightweight, then copied my app and requirements for it. Which I generated with the command `python3 -m  pip freeze > requirements.txt`.

```docker
FROM python:3.7-alpine
ADD app/app-server.py /
ADD requirements.txt /
RUN pip install -r requirements.txt
EXPOSE 8000
CMD python app-server.py
```

Steps:
1. Build the image using the docker file above as follows.
```sh
docker build -t python-app-server -f docker/App-Dockerfile .
```
2. Tag the image as latest
```sh
docker tag XXXXXX rmartinezb/python-app-server:lastest
```
3. Push to repository.
```sh
docker push rmartinezb/python-app-server
```
At this point the image is ready to be used.
```sh
docker run -p 80:8000 python-app-server
```

<h1 align="center">
  <br>
      <img src="/img/posts/201910/test-app.png" alt="Test APP" loading="lazy">
  <br>
</h1>


![]()

## Proxy Server

The application will accept HTTP requests on port a configurable port and route traffic to services that are also configurable.

Functions/Classes:
* configure_error_logging: configure_error_logging: Uses the logging liberty and logs all events to `/tmp/proxy/event.log`
```
def configure_error_logging():
    logger.setLevel(logging.DEBUG)
    # Format for our loglines
    formatter = logging.Formatter("[%(asctime)s] - %(name)s - %(levelname)s - %(message)s")
    # Setup console logging
    ch = logging.StreamHandler()
    ch.setLevel(logging.DEBUG)
    ch.setFormatter(formatter)
    logger.addHandler(ch)
    directory = "/tmp/proxy/"
    if not os.path.exists(directory):
        os.makedirs(directory)

    LOG_FILENAME = directory+"event.log"
    
    # Setup file logging as well
    fh = logging.FileHandler(LOG_FILENAME)
    fh.setLevel(logging.DEBUG)
    fh.setFormatter(formatter)
    logger.addHandler(fh)
```
* ProxyHTTPRequestHandler/do_get: Handle for HTTP Get Requests. It load balances traffic across multiple services/nodes.
```python
class ProxyHTTPRequestHandler(BaseHTTPRequestHandler):
    protocol_version = 'HTTP/1.1'
   

    def do_GET(self, body=True):

            try:
                url = 'https://{}{}'.format(hostname, self.path)

                s =roundRobinService()
                o = roundRobinOrigin(s)
               
                self.server= ""
                self.server_version = ""
                self.sys_version = ""

                endpoint="http://{}:{}{}".format(nodes[s][2][o]['address'],nodes[s][2][o]['port'],self.path)
                
               
                http = requests.Session()
                resp = http.get(endpoint, verify=False) 
                self.send_response_only(resp.status_code)
                self.send_header("Host","{}-{}-{}".format(nodes[s][0],nodes[s][2][o]['address'],nodes[s][2][o]['port']))
                for k,v in resp.headers.items():
                    self.send_header(k,v)
                self.end_headers()
                             
                self.wfile.write(resp.text.encode())
                self.wfile.flush() #actually send the response if not already done.
                self.close_connection= 1
                logger.error("{} - {} - {} ".format(self.address_string(),self.requestline,200))
            except Exception as e:

                self.log_error("Request got ab error out: %r", e)
                logger.error("{} - {} - {} ".format(self.address_string(),self.requestline,200))
                self.close_connection = 1
```
* load_proxy_config: Reads/Opens `config.yaml` configuration file and converts it to a dict. 
```python
def load_proxy_config(config_file):
    with open(config_file, 'r') as stream:
        try:
            return yaml.safe_load(stream)
        except yaml.YAMLError as exc:
            print(exc)
```
* findServices: Finds services within yaml config file and passes to `getOrigins()` to find nodes to route traffic.
```python
def findServices():
    for s in config['proxy']['services']:
        if s['name'] == service:
            return s
    return None

```
```python
def getOrigins():
    global nodes
    services = config['proxy']['services']
    for s in services:
        o=[]
        for h in s['host']:
            i={}
            i['address']=h['address']
            i['port']=h['port']
            o.append(i)
        temp=[s['name'],-1,o]
        nodes.append(temp)
```
* roundRobinService: Uses RoundRobin to route to services and onces service is selected calls `roundRobinOrigin()` to select node.
* roundRobinOrigin: Uses Round Robin to route to multiple nodes.
```python
def roundRobinService():
    
    global n
    n += 1
    return (n% len(nodes))

def roundRobinOrigin(i):
    
    global n
    nodes[i][1] += 1
    return (n% len(nodes[i][2]))
```
### Deploy Proxy Server Docker Image
Again, I need to deploy this application to Kubernetes we need to create an image that we will later deploy in our cluster, same steps as the app server

To do this I created my image with this [dockerfile](https://github.com/roymartinezblanco/Kubernetes-Reverse-Proxy-Deployment/blob/master/docker/Proxy-Dockerfile) (All Docker files can be found under the /docker/ directory.). requirements.txt`.

```docker
FROM python:3.7-alpine
ADD proxy/proxy-server.py /
ADD requirements.txt /
ADD proxy/config.yaml /
ADD proxy/bootstrap.sh /
RUN pip install -r requirements.txt
EXPOSE 8888
CMD sh bootstrap.sh
```
If you notice I have a `bootstrap.sh` script that I'm executing. This is because I don't want the proxy server to start without first modifying its configuration. This script creates an infinite loop to make the server staying up and running.

```sh
#!/bin/bash

while :;do 
        sleep 300
done
```
Steps:
1. Build the image using the docker file above as follows.
```sh
docker build -t python-proxy-server -f docker/Proxy-Dockerfile .
```
2. Tag the image as latest
```sh
docker tag XXXXXX rmartinezb/python-proxy-server:lastest
```
3. Push to repository.
```sh
docker push rmartinezb/python-proxy-server
```
At this point the image is ready to be used.


# Deployment
The deployment of this project I used Kubernetes and this is where a lot my time was spent. Not because it is complicated, but because there is a lot to learn.

Components used:
* [kubectr](https://kubernetes.io/docs/tasks/tools/install-kubectl/): 
* [minikube](https://kubernetes.io/esen/docs/tasks/tools/install-minikube/#instalar-minikube)
* [helm](https://helm.sh/)

In my case I used brew to install all of these components, the issue I was was with helm since it got installed with version 1.6 and I faced issues with `tiller` not getting installed.
```
Error: error installing: the server could not find the requested resource
```
This was resolve by a form, which I sadly lost the link that I wanted to share.

```sh
helm init --override spec.selector.matchLabels.'name'='tiller',spec.selector.matchLabels.'app'='helm' --output yaml | sed 's@apiVersion: extensions/v1beta1@apiVersion: apps/v1@' | kubectl apply -f -
```

Once `Helm` was up and running I was able to create the helm chart template within my project:
```sh
helm create chart # Not a very creative name :)
```

<h1 align="center">
  <br>
      <img src="/img/posts/201910/dir.png" loading="lazy">
  <br>
</h1>

# Deployement

## Helm

First we start with deploying the base app/backend, here as said before we have 3 replicas and we are adding what image we will be using to create or app serve and he `TCP` port that the app will listen to. The `replicas` spec is what enables Kubernetes to know our desired state, meaning if a `replica` fails it will create 1 to meat the 3/3 that is configured below.

```yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-app-server
spec: 
  replicas: 3
  selector:
    matchLabels:
      app: python-app-server
  template:
    metadata:
      labels:
        app: python-app-server
    spec:
      containers:
      - name: python-app-server
        image: rmartinezb/python-app-server:lastest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
```

Also create a very similar deployment for the Proxy server.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-proxy-server
spec: 
  replicas: 1
  selector:
    matchLabels:
      app: python-proxy-server
  template:
    metadata:
      labels:
        app: python-proxy-server
    spec:
      containers:
      - name: python-proxy-server
        image: rmartinezb/python-proxy-server:lastest
        imagePullPolicy: Always
        ports:
        - containerPort: 8999
```
### Services

Services is what enables us to define how we will connect to the pods and be the front for them.

 The `port` is to what the service will be listening and then forwarding to the `targetport`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-app-1
spec:
  selector:
    app: python-app-server
  ports:
    - name: main
      protocol: TCP
      port: 8081
      targetPort: 8000
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-proxy
spec:
  selector:
    app: python-proxy-server
  ports:
    - port: 8888
      protocol: TCP
      targetPort: 8888
```
### Ingress

Once configured the services we need to add a way to communicate with the services. The `ingress` configured below is simple, we are opening `port 80` and sending the traffic to the `proxy-service` via `port 8888`.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-proxy
  annotations:
    http.port: "80"
spec: 
  backend: 
    serviceName: service-proxy
    servicePort: 8888

```

At this point I have everything to create my `helm chart installation`.  
```sh
helm install chart/
```
<h1 align="center">
  <br>
      <img src="/img/posts/201910/helm-install.png)" alt="Helm install" loading="lazy">
  <br>
</h1>

![](

Just like that everything was created. Now we need to configuring the proxy server. To do this, we need to connect to the proxy pod with the following command:
```sh
kubectl exec -it python-proxy-server-xxxxxxx -- /bin/sh
```

Now we need to modify the config.yaml file with the IP from the previous output.
```yaml
proxy:
  listen:
    address: "0.0.0.0"
    port: 8888
  services:
    - name: service-app-1
      host:
      - address: "10.99.246.53"
        port: 8081
    - name: service-app-2
      host:
      - address: "10.98.58.70"
        port: 8081
```
And our last step is to execute the proxy server.
```sh
python3 server-proxy.py
```

# Testing

Now we have everything up and configured but I also wanted to share how I tested my Project.


We will be running two commands, one to see a response from the APP's and another to test the Deployment strategy by failing a server. The requests from my machine will be done to the Kubernetes IP and because the are sending the request to port 80 our ingress and services will route our request.

```
http http://192.168.99.101/ --print=hb
http http://192.168.99.101/fail --verify=no --print=hb
```

<h1 align="center">
  <br>
      <img src="/img/posts/201910/health.gif)" alt="Error Prevention" loading="lazy">
  <br>
</h1>




# Supporting Documentation:
* https://kubernetes.io/blog/2019/07/23/get-started-with-kubernetes-using-python/

* https://matthewpalmer.net/kubernetes-app-developer/articles/guide-install-kubernetes-mac.html
* https://runnable.com/docker/python/dockerize-your-python-application
* https://kubernetes.io/docs/tasks/tools/install-minikube/
* https://kubernetes.io/docs/home/

