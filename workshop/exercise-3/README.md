# Deploy the Guestbook app with Istio Proxy

The Guestbook app is a sample app for users to leave comments. It consists of a web front end, Redis master for storage, and a replicated set of Redis slaves. We will also integrate the app with Watson Tone Analyzer which detects the sentiment in users' comments and replies with emoticons.

![](../README_images/istio1.jpg)

### Download the Guestbook app
1.  Clone the Guestbook app into the `workshop` directory.

    ```shell
    git clone https://github.com/IBM/guestbook.git
    ```

2.  Navigate into the app directory.

    ```shell
    cd guestbook/v2
    ```

### Enable the automatic sidecar injection for the default namespace
In Kubernetes, a sidecar is a utility container in the pod, and its purpose is to support the main container. For Istio to work, Envoy proxies must be deployed as sidecars to each pod of the deployment. There are two ways of injecting the Istio sidecar into a pod: manually using the istioctl CLI tool or automatically using the Istio sidecar injector. In this exercise, we will use the automatic sidecar injection provided by Istio.

1.  Annotate the default namespace to enable automatic sidecar injection:
    
    ``` shell
    kubectl label namespace default istio-injection=enabled
    ```
    
2.  Validate the namespace is annotated for automatic sidecar injection:
    
    ``` shell
    kubectl get namespace -L istio-injection
    ```
    
    Sample output:
    ``` shell
    NAME             STATUS   AGE    ISTIO-INJECTION
    default          Active   271d   enabled
    istio-system     Active   5d2h
    ...
    ```

### Create a Redis database
The Redis database is a service that you can use to persist the data of your app. The Redis database comes with a master and slave modules.

1.  Create the Redis controllers and services for both the master and the slave.

    ``` shell
    kubectl create -f redis-master-deployment.yaml
    kubectl create -f redis-master-service.yaml
    kubectl create -f redis-slave-deployment.yaml
    kubectl create -f redis-slave-service.yaml
    ```

2.  Verify that the Redis controllers for the master and the slave are created.

    ```shell
    kubectl get deployment
    ```
    Output:
    ```shell
    NAME           READY   UP-TO-DATE   AVAILABLE   AGE
    redis-master   1/1     1            1           2m16s
    redis-slave    2/2     2            2           2m15s
    ```

3.  Verify that the Redis services for the master and the slave are created.

    ```shell
    kubectl get svc
    ```
    Output:
    ```shell
    NAME           TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
    redis-master   ClusterIP      172.21.85.39    <none>          6379/TCP       5d
    redis-slave    ClusterIP      172.21.205.35   <none>          6379/TCP       5d
    ```

4.  Verify that the Redis pods for the master and the slave are up and running.

    ```shell
    kubectl get pods
    ```
    Output:
    ```shell
    NAME                            READY     STATUS    RESTARTS   AGE
    redis-master-4sswq              2/2       Running   0          5d
    redis-slave-kj8jp               2/2       Running   0          5d
    redis-slave-nslps               2/2       Running   0          5d
    ```

## Install the Guestbook app

1.  Deploy the Guestbook app on to the Kubernetes cluster. Deploy both the v1 and v2 versions of the app:

    ```shell
    kubectl apply -f ../v1/guestbook-deployment.yaml
    kubectl apply -f guestbook-deployment.yaml
    ```

    These commands deploy the Guestbook app on to the Kubernetes cluster. Since we enabled automation sidecar injection, these pods will be also include an Envoy sidecar as they are started in the cluster. Here we have two versions of deployments, a new version (`v2`) in the current directory, and a previous version (`v1`) in a sibling directory. They will be used in future sections to showcase the Istio traffic routing capabilities.

2.  Create the guestbook service.

    ```shell
    kubectl create -f guestbook-service.yaml
    ```

3.  Verify that the service was created.

    ```shell
    kubectl get svc
    ```
    Output:
    ```shell
    NAME           TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
    guestbook      LoadBalancer   172.21.36.181   169.61.37.140   80:32149/TCP   5d
    ...
    ```

4.  Verify that the pods are up and running.

    ```shell
    kubectl get pods
    ```
    Sample output:
    ```shell
    NAME                            READY   STATUS    RESTARTS   AGE
    guestbook-v1-98dd9c654-dz8dq    2/2     Running   0          30s
    guestbook-v1-98dd9c654-mgfv6    2/2     Running   0          30s
    guestbook-v1-98dd9c654-x8gxx    2/2     Running   0          30s
    guestbook-v2-8689f6c559-5ntgv   2/2     Running   0          28s
    guestbook-v2-8689f6c559-fpzb7   2/2     Running   0          28s
    guestbook-v2-8689f6c559-wqbnl   2/2     Running   0          28s
    redis-master-577bc6fbb-zh5v8    2/2     Running   0          4m47s
    redis-slave-7779c6f75b-bshvs    2/2     Running   0          4m46s
    redis-slave-7779c6f75b-nvsd6    2/2     Running   0          4m46s
    ```

    Note that each guestbook pod has 2 containers in it. One is the guestbook container, and the other is the Envoy proxy sidecar.

### Use Watson Tone Analyzer
Watson Tone Analyzer detects the tone from the words that users enter into the Guestbook app. The tone is converted to the corresponding emoticons.

We've created a shared Tone Analyzer service for you to use for this lab. Refer back to the API_KEY and URL given by the "Grant Cluster" app.

1.  Edit `analyzer-deployment.yaml` with `vi`, a similar editor, or the `sed` commands below:
    * Replace `YOUR_API_KEY` with the API key from Grant Cluster
    * Replace `YOUR_URL` with `https://api.us-south.tone-analyzer.watson.cloud.ibm.com/instances/4cdbea2d-9236-4f58-8a43-1280bfbdfb3f`
    * Save the file.

    > You can do also do this with the sed command:

    ```
    sed -i 's/YOUR_API_KEY/{replace_with_api_key}/g' analyzer-deployment.yaml
    ```

    ```
    sed -i 's+YOUR_URL+https://api.us-south.tone-analyzer.watson.cloud.ibm.com/instances/4cdbea2d-9236-4f58-8a43-1280bfbdfb3f+g' analyzer-deployment.yaml
    ```

2.  Run `cat analyzer-deployment.yaml` and make sure the values under `env:` are all properly set.

    For example:
    ```
            ...
            env:
            - name: VCAP_SERVICES_TONE_ANALYZER_API_KEY
            value: <you should see your API key here>
            - name : VCAP_SERVICES_TONE_ANALYZER_TOKEN_ADDRESS
            value: "https://iam.bluemix.net/identity/token"
            - name: VCAP_SERVICES_TONE_ANALYZER_SERVICE_API
            value: https://api.us-south.tone-analyzer.watson.cloud.ibm.com/instances/4cdbea2d-9236-4f58-8a43-1280bfbdfb3f
            ...
    ```

3.  Deploy the analyzer pods and service, using the `analyzer-deployment.yaml` and `analyzer-service.yaml` files found in the `guestbook/v2` directory. The analyzer service talks to Watson Tone Analyzer to help analyze the tone of a message.

    ```shell
    kubectl apply -f analyzer-deployment.yaml
    kubectl apply -f analyzer-service.yaml
    ```

Great! Your guestbook app is up and running. In Exercise 4, you'll be able to see the app in action by directly accessing the service endpoint. You'll also be able to view Telemetry data for the app.
