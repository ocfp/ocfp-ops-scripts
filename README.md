# OCFP Ops Scripts

Operations Scripts supplimenting the OCFP reference architecture.

There is now an end to end contract in place for the Open(source) Cloud Foundry Platform reference architecture.

## Reference Architecure Contract Process

We perform our standard terraform dance,
```sh
terraform init
terraform validate
terraform apply
```

The resources are created, data is queried, and outputs are placed into vault paths according to the contract, which is effectively `{mount-path}/tf/{env-path}`

As bosh is nearly entirely responsible for interacting with the iaas this most often will look something like `secret/tf/ocfp/mgmt/us/east/1/bosh`.

Once that is complete we then initialize the databases for the deployment (if any) that were provisioned by terraform.
```sh
ocfp init-pg kit bosh region us-east-1 iaas aws env ocfp-mgmt-us-east-1
```

This command will
- Create all required databases for the specified `kit` against the database server provided from terrraform via the contract paths.
- Place the individiual database connection information into the genesis env vault path to be picked up by the kit's `ocfp` feature.

Once the databases have been initialized we can generate the initial genesis environment by running something along the lines of the following.
For a management bosh environment,
```sh
cd ~/deployments/bosh
ocfp init-env kit bosh region us-east-1 iaas aws env ocfp-mgmt-us-east-1
```
For a non mgmt bosh env:
```sh
cd ~/deployments/bosh
ocfp init-env kit bosh region us-east-1 iaas aws env ocfp-us-east-1 bosh ocfp-mgmt-us-east-1
```

Note that all other kits require the bosh director parameter to be given.
```sh
cd ~/deployments/cf
ocfp init-env kit cf region us-east-1 iaas aws env ocfp-us-east-1 bosh ocfp-mgmt-us-east-1
```

In any case the  command will generate an initial environment file with the same name as the `env` parameter, such as `ocfp-mgmt-us-east-1.yml` or `ocfp-us-east-1.yml` from the examples above, in the current working directory.

Next we ensure that the `ocfp` feature is enabled for the env file, which it should be present by default when using the `init-env` command above.

Finally we enhance and extend the env file as needed for the environment and follow the standard genesis approach from that point on:
```
genesis add-secrets
genesis check-secrets
genesis deploy
```

Of course,  for bosh we also have to add `cloud`, `runtime`, and `cpi` configs,
```
ocfp init-config config <type>...
```

## Contract

As mentioned above, the contract between terraform and the scripts & kits is that it will emit the information to: `{mount-path}/tf/{env-path}/` where the `genesis deploy` would normally go to `{mount-path}/{env-path}`.  

NOTE: Since it is BOSH that deals with the iaas, the keys below will nearly all be nested under the `bosh` env path.

The `ocfp` kit feature assumes this and reach directly for them at that contracted path.

The contract at the tf/env path is as follows:  

### Terraform <> Vault

*Definitions*

- `{mount-path}/tf/{env-path}` is denoted as `{tf-env-path}`
- `{mount-path}/{env-path}` is denoted as `{env-path}`

*Asssumptions*

- The context is , unless otherwise indicated.

The terraform vault contract is as follows.

#### Bastions

```
/bastions/{a-z}:ip
```
provides the ip addresses of one or more bastions for the environment.
Bastions are labeled a-z to avoid indexing wars :).

Example: `bastions/a:ip` would contain an IPv4 string such as `10.0.0.253`

Side note, `jumpbox` is deployed via BOSH whereas `bastion` is via terraform.

#### Blobstores

```
/bosh/blobstores/{bosh} # mgmt
/bosh/blobstores/{bosh,app_packages,buildpacks,droplets,resources,shield} # ocf
```

contains information for any blobstores for the env,
Each blobstore contains the following keys,
```
/bosh/blobstores/{name}:{name,arn,region,kms_key_id,kms_key_arn}
```

#### Databases

PostgreSQL databases which are provided by the iaas and are created by terraform
for each env type will be:
```
/dbs/{bosh,concourse} # mgmt
/dbs/{bosh,cf,autoscaler,scheduler,stratos} # ocf
```
Keys are:
```
/dbs/{name}:{hostname,postgres_username,postgres_password}
```
where `hostname` is for the PostgreSQL database server itself and 
`postgres_username`, `postgres_password` contain the postgres admin user login
credentials. These credentials are used later within the reference architecture
to initialize the databases for the `hostname`.

Each database will have corresponding databases created within it by the initialization
process. For example, a `bosh` database after initialization will contain the 
following keys:
```
/dbs/bosh:{hostname,postgres_username,postgres_password,bosh_username,bosh_password,credhub_username,credhub_password,uaa_username,uaa_password}
```

It is assumed with the reference architeture that SSL is enabled and validated
fully. In order to support this the contract also expects that the CA Certificate
for the database systems is placed at:
```
{tf-base-path}}/certs/dbs:ca
```

TODO: Move /certs within each {tf-env-path}, copies are OK, this way in case one
envs certs differ it can easily be overridden.

#### Env

Contains information relevant to the environment's naming and pathing.

This enables downstream to leverage these without having to compute them.

```
/env:{name,path,tf_prefix}
```

#### IAM

```
bosh/iam/{bosh,s3}:{access_key,secret_key}
```
contains the required IAM creds used by env's bosh director in order to interact
with the IaaS and Blobstore.

#### SSH Keys

BOSH Directores require a set of SSH Keypair's for use with the IaaS. Terraform
will generate these and place their information at:
```
bosh/keys/bosh:{public,private,public_pem,private_pem,keypair_name,keypair_type,keypair_id,keypair_arn,keypair_fingerprint}
```
These are then picked up by the `bosh` genesis kit when it uses the `ocfp` feature.

#### Load Balancers

`lbs/` contains the env's load balancer information

#### Network Information

`net/` contains the bosh director's ip, dns, gatweay

#### IaaS Information

```
iaas/kms:arn
```
contains the default kms key arn

```
iaas/region:{name,endpoint}
```
contains the `name` and `endpoint` for the iaas's region

```
iaas/sgs/{default,mgmt,ocf}:id
```
contiains the `default`, `mgmt`, and `ocf` security group ids.

```
iaas/subnets/ocfp/{0,1,2}:{id,cidr_block,cidr_prefix,ip_0,ip_n, availability_zone}
``` 
for the `0`,`1`, and `2` subnets contains the:
- iaas `id`
- `cidr_block` in cidr representation
- `cidr_prefix` first three components of the ipv4 representation
- `ip_0` is the first usable ip address of the subnet's cidr
- `ip_n` is the first usable ip address of the subnet's cidr
- `availability_zone`  is the iaas's specific az of the subnet

```
iaas/vpc:{id,arn,cidr_block}
``` 
contains the vpc's `id`, `arn`, and `cidr_block` in ipv4 cidr format.

### OCFP Init Contract

The `ocfp init-*` components of the contract, which follow `terraform apply`
outputs into vault are as follows.

#### Initialize Databases

`init-pg` parses the cli args given and sets env vars accordingly and then calls the pg 
init script  which in turn  initializes the database system creating 
databases relevant to the specified kit, users, and extensions and finally 
updating `{env-path}` secrets in vault.

An example of using it for an AWS based bosh env would be:
```
ocfp init-pg kit bosh env ocfp-mgmt-us-east-1 region us-east-1 iaas aws
```

#### Initialize Deployment Env File

`init-env` for renders the environment template file for the specified kit and parameters. 

For example the [bosh env template](https://github.com/starkandwayne/ocfp-ops-scripts/blob/main/templates/env/bosh) generates the env file, reflecting on the env name for type `mgmt` or `ocf` and related initial features.


So for example to initialize a Sandbox (sbx) deployment env file which will be 
deployed from a Management (mgmt) bosh director,

```
ocfp init-env kit bosh env ocfp-sbx-us-east-1 region us-east-1 iaas aws bosh ocfp-mgmt-us-east-1
```

#### Initialize BOSH Config

`init-config` renders the config templates for the specified config type and iaas.

```
ocfp init-config type cloud scale prod kit bosh iaas aws env ocfp-mgmt-us-east-1
```

To add the relevant networking information to the mgmt cloud config,
```
ocfp init-config type cloud scale prod kit bosh iaas aws env ocfp-mgmt-us-east-1 dep ocfp-sbx-us-east-1
```

Special note, if a different vault prefix, say `secret/dev/sbx` for the sbx env,
is required per-environment this can be overidden as follows,
```
ocfp init-config type cloud scale prod kit bosh iaas aws env ocfp-mgmt-us-east-1 dep ocfp-sbx-us-east-1%secret/dev/sbx
```

