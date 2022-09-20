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
-   `bastions/` to get the IP info for any bastions
-   `blobstores/` contains information for any blobstores for the env, for example the bosh blobstores `arn`,`name`,`region`, and kms key info.
-   `dbs/` to get the (rds) database information for the env
-   `env` contains the env name, path and tf prefix
-   `iam/` contains the bosh & s3 IAM creds used by env bosh directors
-   `keys/` contains the terraform generated bosh ssh keypairs (these should be picked up by the kit, if not it's a bug to be fixed)
-   `lbs/` contains the env's load balancer information
-   `net/` contains the bosh director's ip, dns, gatweay
-   `iaas/kms` contains the default key arn
-   `iaas/region` contains the name and endpoint,
-   `iaas/sgs` contiains the `default`, `mgmt`, and `ocf` security group ids
-   `iaas/subnets/ocfp` contains the `id`, `cidr_block`, and `availability_zone` for the `0`,`1`, and `2` subnets.
-   `iaas/vpc` contains the vpc's `id`, `arn`, and `cidr_block`.

As for the `ocfp init` components of the contract, they reside here [https://github.com/starkandwayne/ocfp-ops-scripts](https://github.com/starkandwayne/ocfp-ops-scripts) 

The `ocfp` script for env initialization is documented in the usage currently.

By default `ocfp init` will do `init-pg` followed by `init-env` .

`init-pg` parses the cli args given and sets env vars accordingly and then calls the pg init script  
[https://github.com/starkandwayne/ocfp-ops-scripts/blob/main/templates/pg/init](https://github.com/starkandwayne/ocfp-ops-scripts/blob/main/templates/pg/init)  
Which initializes the databases given by TF and then sets the secrets in the vault env path to be picked up by the `ocfp` feature of the kit for database info. 

`init-env` for renders the environment template file for the specified kit and parameters. For example the [bosh env template](https://github.com/starkandwayne/ocfp-ops-scripts/blob/main/templates/env/bosh) generates the env file, reflecting on the env name for type `mgmt` or `ocf` and related initial features.

`init-config` is currently being implemented and needs to be tested & debugged. It renders the env templates for the specified [config type](https://github.com/starkandwayne/ocfp-ops-scripts/tree/main/templates/configs/cloud/aws)

