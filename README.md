# Basic Example of deploying Supabase to an Openshift Cluster

Note this repo is modified version of [this example](https://github.com/supabase-community/supabase-kubernetes/blob/main/charts/supabase/README.md), updated to work on Kubernetes.

Currently this is still a WIP but this should allow for the deployment of Supabase on an Openshift cluster.

## Things to Note

The Kong API layer has authentication disabled (although the other components all seem to have their own authentication layer). This can be reenabled from the `configs/kong.yaml` if required.

The db and meta component I have using a custom image which sources can be found under the `Dockerfiles` folder. Meta was due to a permissions issue when deploying on Openshift and DB was because my app required a Postgres extension that was not included.

## My Environment

My group is currently using a [ROSA(Managed Openshift on AWS)](https://www.redhat.com/en/technologies/cloud-computing/openshift/aws?sc_cid=7013a000003Sfj2AAC&gad_source=1&gclid=EAIaIQobChMI7eyN5tzGhQMVzE5HAR3b6A0ZEAAYASAAEgIuZPD_BwE&gclsrc=aw.ds) cluster. And the application is being deployed through ArgoCD.

## Environment Setup

```bash
 
# JWT Values

# Secret used by AUth to sign user JWTs
export SECRET=$(openssl rand -base64 32 | tr -dc 'a-zA-Z0-9')

# Secret used by REALTIME for signing info
export COOKIE_SECRET=$(openssl rand -base64 64)

# Long Lived JWT used as an Anonymous key
export ANONYMOUS_KEY=$(podman run -e SECRET docker.io/bitnami/jwt-cli:latest encode --secret "$SECRET" --exp=$(date -d '+10 years' +%s) --payload role=anon --iss supabase)

# Long Lived JWT used as a service key
export SERVICE_KEY=$(podman run -e SECRET docker.io/bitnami/jwt-cli:latest encode --secret "$SECRET" --exp=$(date -d '+10 years' +%s) --payload role=service_role --iss supabase)

# Create JWT secret
oc create secret generic supabase-jwt \
--from-literal=anonKey="${ANONYMOUS_KEY}" \
--from-literal=serviceKey="${SERVICE_KEY}" \
--from-literal=secret="${SECRET}" \
--from-literal=secretCookie="${COOKIE_SECRET}"

export SMTP_PASSWORD=$(openssl rand -base64 24 | tr -dc 'a-zA-Z0-9')

# Create SMTP secret
oc create secret generic supabase-smtp \
--from-literal=username="your-mail@example.com" \
--from-literal=password="$SMTP_PASSWORD"

export DB_PASSWORD=$(openssl rand -base64 24 | tr -dc 'a-zA-Z0-9')
  
# Create DB secret
oc create secret generic supabase-db \
--from-literal=username="postgres" \
--from-literal=password="$DB_PASSWORD"

# This creates the pull secret that allows our pipelines to pull from our gitlab repository
oc create secret generic gitlab-pull-secret \
--from-literal=username=data-migration-token \
--type=kubernetes.io/basic-auth \
--from-literal=password=<password> 

oc annotate secret gitlab-pull-secret tekton.dev/git-0=https://gitlab.consulting.redhat.com

oc secrets link pipeline gitlab-pull-secret

```

## Required Changes

We are deploying using the `values-example.yaml` file. This is going to require some changes, the ones I know of include:

1. Changing all of the `https://supabase-kong-<NAMESPACE>.apps.<GUID>.p1.openshiftapps.com/` to point to your application's Kong ingress point
2. Storage for DB will need to reflect your storage classes (eventually postgres should probably live offcluster)
3. I think that is it.. but I am probably missing stuff