# gitwebhook

A simple git webhook handler that demonstrates interacting with
github.
[Modeled after GCF example](https://cloud.google.com/community/tutorials/github-auto-assign-reviewers-cloud-functions)

## Prerequisites

1. [Install Knative Serving](https://github.com/knative/docs/blob/master/install/README.md)
1. Install [docker](https://www.docker.com/)

## Setup

Build the app container and publish it to your registry of choice:

```shell
REPO="gcr.io/<your-project-here>"

# Build and publish the container, run from the root directory.
docker build \
  --tag "${REPO}/serving/samples/gitwebhook-go" \
  --file=serving/samples/gitwebhook-go/Dockerfile .
docker push "${REPO}/serving/samples/gitwebhook-go"

# Replace the image reference with our published image.
perl -pi -e "s@github.com/knative/docs/serving/samples/gitwebhook-go@${REPO}/serving/samples/gitwebhook-go@g" serving/samples/gitwebhook-go/*.yaml

# Deploy the Knative Serving sample
kubectl apply -f serving/samples/gitwebhook-go/sample.yaml
```

## Exploring

Once deployed, you can inspect the created resources with `kubectl` commands:

```shell
# This will show the Route that we created:
kubectl get route -o yaml

# This will show the Configuration that we created:
kubectl get configurations -o yaml

# This will show the Revision that was created by our configuration:
kubectl get revisions -o yaml

```

To make this service accessible to github, we first need to determine its ingress address
(might have to wait a little while until `EXTERNAL-IP` gets assigned):
```shell
watch kubectl get svc knative-ingressgateway -n istio-system
NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                                      AGE
knative-ingressgateway   LoadBalancer   10.23.247.74   35.203.155.229   80:32380/TCP,443:32390/TCP,32400:32400/TCP   2d
```

Once the `EXTERNAL-IP` gets assigned to the cluster, you need to assign a DNS name for that IP address.
[Using GCP DNS](https://support.google.com/domains/answer/3290350)

So, you'd need to create an A record for demostuff.aikas.org pointing to 35.202.30.59.

Then you need to go to github and [set up a webhook](https://cloud.google.com/community/tutorials/github-auto-assign-r
eviewers-cloud-functions).
For the Payload URL however, use your DNS entry you created above, so for my example it would be:
http://demostuff.aikas.org/

Create a secret that has access to the tokens. Take the Secret you used for the webhook
(secretToken) and the generated access token (accessToken) (as per the above  webhook)

```shell
echo -n "your-chosen-secret-token" > secretToken
echo -n "github-generated-access-token" > accessToken
kubectl create secret generic githubsecret --from-file=./secretToken --from-file=./accessToken
```

Then create a PR for the repo you configured the webhook for, and you'll see that the Title
will be modified with the suffix '(looks pretty legit)'

## Cleaning up

To clean up the sample service:

```shell
kubectl delete -f serving/samples/gitwebhook-go/sample.yaml
```