# vault-gcp-iam
The [Vault GCP secrets engine](https://developer.hashicorp.com/vault/docs/secrets/gcp) allows Vault users to generate IAM service account credentials with a given set of permissions and a set lifetime, without needing a service account of their own. This repo show you how to use the secrets backend to generate credentials to authorize a call to GCP.

## Start Vault Server in dev
- server
```
vault server -dev
```

- client 
```
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_DEV_ROOT_TOKEN_ID="hvs.3hAN2hxY4wnD3C8A3o8Ny8Yb" # root token
```

## Setup GCP Secrets Engine
- Enable the Google Cloud secrets engine:
```
# By default, the secrets engine will mount at the name of the engine. To enable the secrets engine at a different path, use the -path argument.
vault secrets enable gcp
```

- Configure the secrets engine with account credentials, [Google Cloud credentials ref](https://developer.hashicorp.com/vault/docs/secrets/gcp#authentication)
```
vault write gcp/config credentials=/Users/rich/.config/gcloud/application_default_credentials.json
```

### [Rolesets](https://developer.hashicorp.com/vault/docs/secrets/gcp#rolesets)
A roleset consists of a Vault managed GCP Service account along with a set of IAM bindings defined for that service account. The name of the service account is generated based on the time of creation or update. You should not depend on the name of the service account being fixed and should manage all IAM bindings for the service account through the [bindings](https://developer.hashicorp.com/vault/docs/secrets/gcp#bindings) parameter when creating or updating the roleset.

### [Bindings](https://developer.hashicorp.com/vault/docs/secrets/gcp#bindings)
Bindings define a list of resources and the associated IAM roles on that resource. Bindings are used as the binding argument when creating or updating a roleset or static account and are specified in the following format using HCL:
```hcl
resource NAME {
  roles = [ROLE, [ROLE...]]
}
```

- define a binding for project viewer
```hcl
resource "//cloudresourcemanager.googleapis.com/projects/dascs-lab" {
    roles = ["roles/viewer"]
}
```

- Configure a roleset that generates OAuth2 access tokens (preferred)
```
vault write gcp/roleset/my-token-roleset \
    project="dascs-lab" \
    secret_type="access_token"  \
    token_scopes="https://www.googleapis.com/auth/cloud-platform" \
    bindings=@mybindings.hcl
```
- Configure a roleset that generates GCP Service Account keys:
```
vault write gcp/roleset/my-key-roleset \
    project="dascs-lab" \
    secret_type="service_account_key"  \
    bindings=@mybindings.hcl
```

## Generate OAuth2 tokens or service account keys

- To generate OAuth2 tokens
```
vault read gcp/roleset/my-token-roleset/token
```

- To generate service account keys
```
vault read gcp/roleset/my-key-roleset/key
```

## Test GCP Cloud Storage API
- List bucket
```
curl https://storage.googleapis.com/storage/v1/b?project=dascs-lab \
-H 'Authorization: Bearer {TOKEN}'
```

- Create a bucket
```
curl -X POST https://storage.googleapis.com/storage/v1/b?project=dascs-lab \
   -H 'Content-Type: application/json' \
   -H 'Authorization: Bearer {TOKEN}' \
   -d '{"name":"rich-bucket-1234"}'
```