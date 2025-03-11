# Apigee Envoy Gateway Sample
This project is an example for deploying the Apigee Envoy Adapter inside an Envoy Gateway deployment in Kubernetes.

```sh
# step 1 - create secrets for Apigee Envoy config.yaml and application_default_credentials.json
kubectl create secret generic envoy-config -n envoy-gateway-system \
	--from-file=./config.yaml \
	--from-file=/home/USER/.config/gcloud/application_default_credentials.json

# step 2 - deploy apigee remote service into the cluster
kubectl apply -f apigee-deployment.yaml -n envoy-gateway-system

# step 3 - TODO WIP apply envoy bootstrap configuration to use apigee filters
# get bootstrap config
egctl config envoy-proxy all
# TODO update with apigee filters and apply - https://gateway.envoyproxy.io/v1.0/tasks/operations/customize-envoyproxy/#customize-envoyproxy-bootstrap-config
```