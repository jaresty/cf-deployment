# Using `cf-deployment` With GCP
This document contains IaaS-specific notes and instructions for using `cf-deployment` with GCP. See the main [README](https://github.com/cloudfoundry/cf-deployment/blob/master/README.md) for more general information about `cf-deployment`.

## Environment Setup
1. You'll need to make sure your GCP quotas are large enough. You can request a quota increase and ask for the following increases:
  - 100 CPUs
  - 50 In-use IP addresses
1. Clone the cf-deployment repository
  ```
  git clone git@github.com:cloudfoundry/cf-deployment.git
  ```
1. Terraform environment: https://github.com/cloudfoundry-incubator/cf-gcp-infrastructure/blob/master/README.md
1. Update DNS in Route 53 to contain the name servers made by terraform

1. Generate ssh key for director manifest and add to GCP
  ```
  ssh-keygen -t rsa -f /PATH/TO/DIRECTOR/SSH_KEY -C vcap  -N ""
  paste -d: <(echo vcap) /PATH/TO/DIRECTOR/SSH_KEY.pub > /PATH/TO/DIRECTOR/SSH_KEY.gcp_pub
  gcloud auth activate-service-account --key-file /PATH/TO/GOOGLE_AUTH_JSON
  gcloud config set project PROJECT_ID
  gcloud config set compute/region REGION
  gcloud config set compute/zone ZONE
  gcloud compute project-info add-metadata --metadata-from-file sshKeys=/PATH/TO/DIRECTOR/SSH_KEY.gcp_pub
  ```
  
1. Generate director certificate
  ```
  cf-gcp-infrastructure/deployments/generate-certs.sh director DIRECTOR_IP bosh.ENV_NAME.cf-app.com /TARGET_DIRECTORY/FOR/CERTS
  ```
  
1. Generate Bosh deployment var file containing the following keys:
  ```
  project: PROJECT_ID
  zone: ZONE
  env_name: ENV_NAME
  director_ip: DIRECTOR_IP
  nats_password: SOME_PASSWORD
  postgres_password: SOME_PASSWORD
  blobstore_director_password: SOME_PASSWORD
  blobstore_agent_password: SOME_PASSWORD
  hm_password:  SOME_PASSWORD
  mbus_password: SOME_PASSWORD
  director_cert: contents of /TARGET_DIRECTORY/FOR/CERTS/director.crt
  director_key: contents of /TARGET_DIRECTORY/FOR/CERTS/director.key
  google_cpi_json_key: stringified GOOGLE_AUTH_JSON content
  director_username: BOSH_USERNAME
  director_password: BOSH_PASSWORD
  director_ssh_key_path: /PATH/TO/DIRECTOR/SSH_KEY
  ```
  
1. Deploy bosh. NB: You need the new [BOSH CLI](https://github.com/cloudfoundry/bosh-cli) to run `create-env`.
  ```
  bosh interpolate -l DEPLOYMENT_VAR_FILE --var-errs cf-gcp-infrastructure/bosh/bosh.yml > /dev/null
  bosh create-env --var-file DEPLOYMENT_VAR_FILE cf-gcp-infrastructure/bosh/bosh.yml
  ```
  
1. Save the `bosh-state.json` file now located at `cf-gcp-infrastructure/bosh/bosh-state.json`
1. Upload cloud config
  ```
  export BOSH_USER=BOSH_USERNAME
  export BOSH_PASSWORD=BOSH_PASSWORD
  export BOSH_ENVIRONMENT=DIRECTOR_IP
  export BOSH_CA_CERT=/TARGET_DIRECTORY/FOR/CERTS/rootCA.pem

  bosh log-in
  bosh -n update-cloud-config \
    --var-file=DEPLOYMENT_VAR_FILE \
     cf-gcp-infrastructure/deployments/cloud-config.yml
  ```

## Deploying CF
1. Upload the current stemcell for `cf` by running the command below with the appropriate version number. The version number is specified on the last line of `cf-deployment.yml`.
  ```
  bosh upload stemcell https://bosh.io/d/stemcells/bosh-google-kvm-ubuntu-trusty-go_agent?v=VERSION
  ```
 -  Use instructions in the [main README](https://github.com/cloudfoundry/cf-deployment/blob/master/README.md) from cf-deployment to generate a vars file if you don't already have one.
1. Check that you can generate a final manifest without errors
  ```
  bosh -n interpolate -l CF_DEPLOYMENT_VARS_FILE -o manifest/gcp.yml --var-errs manifest/cf-deployment.yml
  ```
1. Deploy!
  ```
  bosh \
    -n \
    -d "cf" \
    deploy \
    -l CF_DEPLOYMENT_VARS_FILE \
    -o manifest/gcp.yml \
    "manifest/cf-deployment.yml"
  ```
