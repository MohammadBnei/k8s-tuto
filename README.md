# K8s tutorial

## Prerequisites

Make sure you have kubernetes installed and running
```
    kubectl version
```
Should output the version of kubectl (Server & Client)


## WSL
We will install a dashboard to get a GUI
```
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc6/aio/deploy/recommended.yaml
    kubectl get all -n kubernetes-dashboard
```

It is not accessible from outside of the cluster by default, so let's instanciate a temporary proxy
```
    kubectl proxy
```
You can access the dashboard now at this address : 
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

You can't login yet, you need to create a service account
```
    # Create a new ServiceAccount
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: ServiceAccount
    metadata:
        name: admin-user
        namespace: kubernetes-dashboard
    EOF

    # Create a ClusterRoleBinding for the ServiceAccount
    kubectl apply -f - <<EOF
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
        name: admin-user
    roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: admin-user
      namespace: kubernetes-dashboard
    EOF

    # Get the Token for the ServiceAccount
    kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```
Copy the token and paste it into the Dashboard login and press "Sign in"

Voila !

## First basic deployment

Your first deployment will be very simple 
```
    kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
```

See on your dashboard the new deployment, and use the url (verify that the ```kubectl proxy``` is still running) to access the pod :
http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME

That's it ! If you have an published image on a public registery, you can try to deploy it.

To delete the deployment : 
```
    kubectl delete deployment kubernetes-bootcamp
```

## Let's dive into it

### (Optional) Push images to docker registry

To push images to the registry, you must first log into docker
```
    docker login
```
It will ask you you username and password, classic

Then you can build the image and push it into the docker registry
```
    # Put the correct information when there is a $
    docker build . -t $DOCKER_USERNAME/$IMAGE_NAME:($TAG) 
    docker push $DOCKER_USERNAME/$IMAGE_NAME:($TAG) 
```

### User Deployment

You can use the mohammaddocker/k8s-tuto-user-api for the image if you did not build your own.

A deployment is a set of instruction to a desired state. You can create the user-deployment.yml file and copy the next lines :
```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
        name: user
        labels:
            app: user
    spec:
        replicas: 2
        selector:
            matchLabels:
            app: user
        template:
            metadata:
                labels:
                    app: user
            spec:
            containers:
                - name: user
                image: $IMAGE
                env:
                    - name: PG_URI
                      value: $PG_URI
                    - name: JWT_KEY
                      value: $JWT_KEY
                    - name: PORT
                      value: $PORT
                    - name: CLIENT_URL
                      value: $CLIENT_URL
                ports:
                    - containerPort: $PORT
                      name: http
                resources:
                    limits:
                    memory: 512Mi
                    cpu: "0.5"
                    requests:
                    memory: 512Mi
                    cpu: "0.5"
```
Change the value starting with $ with your configuration

You can test the deployment with this command :
```
    kubectl apply -f user-deployment.yml
```

By default, all ip are internalized (Cluster, Node, Port) so you cannot access it from the outside. Verify that the deployment went smooth with the dashboard (or ```kubectl get deployment```), then port forward the pods
```
    kubectl port-forward deployment/user $LOCAL_PORT:$POD_PORT
```

http://localhost:$LOCAL_PORT/api/v1/docs/

### Databses

#### PostgreSQL

Although you can specify outside URL to your database of choice, we will deploy postgres and mongo inside the k8s cluster for the kiff.

We will introduce helm here, which permits simplified complexity management for k8s
```
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
    chmod +x get_helm.sh
    ./get_helm.sh
```

Create the postgres deployment with helm :
```
    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm install pg bitnami/postgresql
```
From the log outputs, copy the internal url of your pg instance and get the password :
```
    kubectl get secret --namespace default pg-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode
```

You can put these information inside the user-deployment and re-apply it, Re-run the port-forwarding option and then you can test the insertion of a new user :
```
    curl -X 'POST' \
        'http://localhost:3000/api/v1/auth/signup' \
        -H 'accept: application/json' \
        -H 'Content-Type: application/x-www-form-urlencoded' \
        -d 'email=test%40test.com&password=test&username=test' \
        | json_pp
```

#### Mongo

Similarly, let's use helm to install Mongo
```
    helm install mgo bitnami/mongodb
```

Get the password and url for later
```
    kubectl get secret --namespace default mgo-mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 --decode
```

### Product Deployment

Inside the product-deployment.yml, put the following
```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: product
    labels:
        app: product
    spec:
    replicas: 2
    selector:
        matchLabels:
        app: product
    template:
        metadata:
        labels:
            app: product
        spec:
        containers:
            - name: product
            image: mohammaddocker/k8s-tuto-product-api
            imagePullPolicy: Always
            env:
                - name: MONGO_URI
                  value: mongodb://root:$MONGO_PASSWORD@$MONGO_URI:27017/
                - name: JWT_KEY
                  value: your-jwt-key
                - name: PORT
                  value: '3000'
            ports:
                - containerPort: 3000
                  name: http
            resources:
                limits:
                memory: 512Mi
                cpu: "0.5"
                requests:
                memory: 512Mi
                cpu: "0.5"
```

Apply the file
```
    kubectl apply -f product-deployment.yml
```

Forward the port and test it (you'll need a token from the user-api)
```
    kubectl port-forward deployment/product 3001:3000
    
    curl -X 'POST' http://localhost:3001/api/v1/products' \
        -H 'Authorization: Bearer $TOKEN' \
        -H 'Content-Type: application/x-www-form-urlencoded' \
        -H 'accept: application/json' \
        --data-raw 'value=36&name=test&vendorId=1' \
        --compressed | json_pp
```

### Payment API

The payment api is special, because it needs to know the network location of the product and user api.

Services are the basic network locator in kubernetes. Let's add one for user and product at the end of each files

```
    # Inside user-deployment.yml
    ---
    apiVersion: v1
    kind: Service
    metadata:
        name: user
    spec:
        selector:
            app: user
        ports:
            - protocol: TCP
            port: 80
            targetPort: 3000

    # Inside product-deployment.yml
    ---
    apiVersion: v1
    kind: Service
    metadata:
        name: product
    spec:
        selector:
            app: product
        ports:
            - protocol: TCP
            port: 80
            targetPort: 3000
```

Apply the two modified files

Let's write the payment deployment:
```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: payment
    labels:
        app: payment
    spec:
    replicas: 2
    selector:
        matchLabels:
        app: payment
    template:
        metadata:
        labels:
            app: payment
        spec:
        containers:
            - name: payment
            image: mohammaddocker/k8s-tuto-payment-api
            imagePullPolicy: Always
            env:
                - name: MONGO_URI
                    value: $MONGO_URI
                - name: JWT_KEY
                    value: your-jwt-key
                - name: PORT
                    value: '3000'
                # The url is the name of the service for each deployment
                - name: USER_API_URL
                    value: http://user/api/v1/
                - name: PRODUCT_API_URL
                    value: http://product/api/v1/
            ports:
                - containerPort: 3000
                name: http
            resources:
                limits:
                memory: 512Mi
                cpu: "0.5"
                requests:
                memory: 512Mi
                cpu: "0.5"
```

As always, forward the port of the deployment and you can test if everything went alright
```
    kubectl port-forward deployment/payment 3002:3000

    curl -X 'POST' 'http://localhost:3002/api/v1/payments' \
        -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwiZW1haWwiOiJ0ZXN0QHRlc3QuY29tIiwiaWF0IjoxNjM5MzQwNTMzLCJleHAiOjE2NDAyMDQ1MzN9.46GGgDsqhKMt0Pa6X5LWMajagWSCY8IioSljbyio8z0' \
        -H 'Content-Type: application/x-www-form-urlencoded' \
        -H 'accept: application/json' \
        --data-raw 'payed=90&buyerId=1&productId=$PRODUCT_ID' \
        --compressed | json_pp
```

That's it ! You've succesfully created deployment and service script in kubernetes.