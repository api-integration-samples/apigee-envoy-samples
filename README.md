# Apigee Envoy Gateway Samples
This project shows how to deploy and use apigee + envoy as a local gateway both in Kubernetes and in Cloud Run.

## Prerequisites
- Install the Apigee Envoy Adapter CLI in your environment.
```sh
# see latest releases here: https://github.com/apigee/apigee-remote-service-cli/releases
VERSION=2.1.4
# download the cli
wget "https://github.com/apigee/apigee-remote-service-cli/releases/download/v2.1.4/apigee-remote-service-cli_${VERSION}_linux_64-bit.tar.gz"
# unpack
tar -xf "apigee-remote-service-cli_${VERSION}_linux_64-bit.tar.gz"
# remove tar
rm "apigee-remote-service-cli_${VERSION}_linux_64-bit.tar.gz"
# optional - copy to /usr/bin to more easily use
sudo mv apigee-remote-service-cli /usr/bin
```
## Deployment

### Local Docker
```sh
# start apigee adapter
docker run --rm -it \
-v $(pwd)/config-apigee.local.yaml:/mnt/config/config.yaml \
-e GOOGLE_APPLICATION_CREDENTIALS=/etc/envoy/application_default_credentials.json \
-v $HOME/.config/gcloud/application_default_credentials.json:/etc/envoy/application_default_credentials.json:ro \
-p 5000:5000 \
google/apigee-envoy-adapter:v2.1.4 -c /mnt/config/config.yaml

# start envoy proxy
docker run --rm -it \
--network host \
-v $(pwd)/config-envoy.local.yaml:/mnt/config/config.yaml \
-e GOOGLE_APPLICATION_CREDENTIALS=/mnt/config/application_default_credentials.json \
-v $HOME/.config/gcloud/application_default_credentials.json:/mnt/config/application_default_credentials.json:ro \
-p 9901:9901 -p 10000:10000 \
envoyproxy/envoy:contrib-dev -c /mnt/config/config.yaml

# call api
curl -i http://localhost:10000/echo -H "x-api-key: test"
```

## Kubernetes
```sh
# set env variables where Apigee X is provisioned
PROJECT_ID=YOUR_PROJECT_ID
APIGEE_ENV=YOUR_APIGEE_ENVIRONMENT
# the host is where the Apigee environment is reachable, i.e. 34-112-147-154.nip.io. you can list all of your apigee hosts with 'apigeecli envgroups list -o $PROJECT_ID -t $(gcloud auth print-access-token)'
APIGEE_HOST=YOUR_APIGEE_HOST

# provision the envoy adapter to Apigee - this only has to be done one time
apigee-remote-service-cli provision --organization $PROJECT_ID --environment $APIGEE_ENV --runtime "https://$APIGEE_HOST" --token $(gcloud auth print-access-token) > config-apigee.local.yaml
# the resulting config-apigee.local.yaml contains the key and configuration information for the envoy adapter.
# change api operation identifier to header
sed -i "/      append_metadata_headers: true/c\      append_metadata_headers: true\n      api_header: x-api-operation" config-apigee.local.yaml

# copy envoy config with example clusters and routes, or bring your own
cp config-envoy.yaml config-envoy.local.yaml
# replace Apigee host in envoy config with your host
sed -i "s/APIGEE_HOST/$APIGEE_HOST/g" config-envoy.local.yaml

# create secrets for all configuration, including local default credentials for testing
kubectl create ns apigee
kubectl create secret generic envoy-config -n apigee \
  --from-file=./config-apigee.local.yaml \
  --from-file=./config-envoy.local.yaml \
  --from-file=/home/USER/.config/gcloud/application_default_credentials.json

# deploy apigee remote service into the cluster
kubectl apply -f apigee-deployment.yaml -n apigee

# call api
curl -i http://SERVICE_IP:8080/echo -H "x-api-key: test"
```