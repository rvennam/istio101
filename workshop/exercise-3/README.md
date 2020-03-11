# Exercise 3 - Deploy the Guestbook app with Istio Proxy

The Guestbook app is a sample app for users to leave comments. It consists of a web front end, Redis master for storage, and a replicated set of Redis slaves. We will also integrate the app with Watson Tone Analyzer which detects the sentiment in users' comments and replies with emoticons.

![](../README_images/istio1.jpg)

### Download the Guestbook app
1.  Clone the Guestbook app into the `workshop` directory.

    ```shell
    git clone -b kubecon2019 https://github.com/IBM/guestbook
    ```

2. Navigate into the app directory.

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

4. Verify that the Redis pods for the master and the slave are up and running.

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

1. Inject the Istio Envoy sidecar into the guestbook pods, and deploy the Guestbook app on to the Kubernetes cluster. Deploy both the v1 and v2 versions of the app:

    ```shell
    kubectl apply -f ../v1/guestbook-deployment.yaml
    kubectl apply -f guestbook-deployment.yaml
    ```

These commands deploy the Guestbook app on to the Kubernetes cluster. Since we enabled automation sidecar injection, these pods will be also include an Envoy sidecar as they are started in the cluster. Here we have two versions of deployments, a new version (`v2`) in the current directory, and a previous version (`v1`) in a sibling directory. They will be used in future sections to showcase the Istio traffic routing capabilities.

2. Create the guestbook service.

    ```shell
    kubectl create -f guestbook-service.yaml
    ```

3. Verify that the service was created.

    ```shell
    kubectl get svc
    ```
    Output:
    ```shell
    NAME           TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
    guestbook      LoadBalancer   172.21.36.181   169.61.37.140   80:32149/TCP   5d
    ```

4. Verify that the pods are up and running.

    ```shell
    kubectl get pods
    ```
    Sample output:
    ```shell
    NAME                            READY     STATUS    RESTARTS   AGE
    guestbook-v1-89cd4b7c7-frscs    2/2       Running   0          5d
    guestbook-v2-56d98b558c-mzbxk   2/2       Running   0          5d
    ```

    Note that each guestbook pod has 2 containers in it. One is the guestbook container, and the other is the Envoy proxy sidecar.


### Use Watson Tone Analyzer
Watson Tone Analyzer detects the tone from the words that users enter into the Guestbook app. The tone is converted to the corresponding emoticons.

Create Watson Tone Analyzer in your own account.

1. Switch to your own account by logging in again.

    ```shell
    ibmcloud login
    ```

1. Choose your own account (NOT IBM).

1. If prompted to choose a region, select us-south.

1. Create Watson Tone Analyzer service in the `default` resource group. 

    ```shell
    ibmcloud resource service-instance-create my-tone-analyzer-service tone-analyzer lite us-south -g default
    ```

    > If the previous command errors, it might be due to your resource group name. Try -g Default rather than default
    
    > See all resource groups by running `ibmcloud resource groups`. If it fails due to the region, try `au-syd` rather than `us-south`.

1. Create the service key for the Tone Analyzer service. This command should output the credentials you just created. You will need the value for **apikey** & **url** later.

    ```shell
    ibmcloud resource service-key-create tone-analyzer-key Manager --instance-name my-tone-analyzer-service
    ```
1. If you need to get the service-keys later, you can use the following command:

    ```shell
    ibmcloud resource service-key tone-analyzer-key
    ``` 

1. Open the web file browser by clicking the Pen icon. 

    ![](../README_images/fileeditor.png)

1. Click on the File Explorer icon to see the available files.

    ![](../README_images/file_explorer.png)

1. Navigate to `istio101/workshop/guestbook/v2/analyzer-deployment.yaml`

![](../README_images/fileeditor2.png)


1. Find the env section near the end of the file. Replace YOUR_API_KEY with the API_KEY provided earlier. YOUR_URL should be edited to be `https://gateway.watsonplatform.net/tone-analyzer/api`. If you created the service in `au-syd`, this URL should instead be: `https://gateway-syd.watsonplatform.net/tone-analyzer/api` Save the file and close the web file browser.


Note: You may have trouble editing this file with some browsers, particularly Edge. If you're having trouble, try clicking the name of the file in the top tab, and then moving your cursor and making edits using the arrow keys and keyboard. `ctrl+c` is copy, and `ctrl+v` is paste.

1.   Deploy the analyzer pods and service, using the `analyzer-deployment.yaml` and `analyzer-service.yaml` files found in the `guestbook/v2` directory. The analyzer service talks to Watson Tone Analyzer to help analyze the tone of a message. Ensure you are still in the `guestbook/v2` directory.

      ```shell
      kubectl apply -f analyzer-deployment.yaml
      kubectl apply -f analyzer-service.yaml
      ```

Great! Your guestbook app is up and running. In Exercise 4, you'll be able to see the app in action by directly accessing the service endpoint. You'll also be able to view Telemetry data for the app.

#### [Continue to Exercise 4 - Telemetry](../exercise-4/README.md)
