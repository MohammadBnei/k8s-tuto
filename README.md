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



