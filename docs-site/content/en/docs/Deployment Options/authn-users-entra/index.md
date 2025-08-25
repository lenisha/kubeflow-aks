---
categories: ["getstarted"]
tags: ["test", "sample", "docs"]
title: "Authenticate Kubeflow users with Custom Password or Entra Id"
linkTitle: "Authenticate Kubeflow users with Custom Password or Entra Id"
date: 2025-08-23
weight: 3
description: >
  Authenticating Kubeflow users on AKS with Custom Password or Entra Id
---

## Background

In this lab, you will update the [Kubeflow vanilla installation](../vanilla-installation) option to configure authentication using either custom users and  passwords or Azure Entra ID.

## Change default password
{{< alert color="warning" >}}⚠️ Warning: Always Update the default password before making Kubeflow deployment accessible from outside the cluster.{{< /alert >}}

To change the default password for the Kubeflow dashboard, you need to update the Dex configuration. 
1. First generate Password/Hashes by following steps described in `kubeflow` docs [using python to generate bcrypt hash](https://github.com/kubeflow/manifests/blob/master/README.md#change-default-user-password). Or for simplicity you can use an online tool like [bcrypt-generator](https://www.bcrypt-generator.com/) to create a new hash.

```bash
pip3 install passlib
python3 -c 'from passlib.hash import bcrypt; import getpass; print(bcrypt.using(rounds=12, ident="2y").hash(getpass.getpass()))'

Password: ***
$2y$12$XXXXXXXXXXXXXXXXXXX
```
2. Delete existing password
```bash
kubectl delete secret dex-passwords -n auth
```
3. Create new password secret
```bash
kubectl create secret generic dex-passwords --from-literal=DEX_USER_PASSWORD='REPLACE_WITH_HASH' -n auth
```
4. Restart the Dex deployment to pick up the new password secret:
```bash
kubectl rollout restart deployment dex -n auth
```

To add more users 
1. update `dex` config map `deployments/vanilla/dex-config-map.yaml` with more entries in user array:

```
    staticPasswords:
    - email: user@example.com
      hashFromEnv: DEX_USER_PASSWORD
      username: user
      userID: "15841185641784"
      # Add more users here
    - email: user2@example.com
        hashFromEnv: DEX_USER2_PASSWORD
        username: user2
        userID: "15841185641785"
```

2. Update `DEX_USER2_PASSWORD` with the new password hash.

```bash
kubectl patch secret dex-passwords -n auth --type='json' -p='[{"op": "replace", "path": "/data/DEX_USER2_PASSWORD", "value":"'$(echo -n 'REPLACE_WITH_HASH' | base64)'"}]'
```
3. Apply config map and restsrt deployment

```bash
kubectl apply -f deployments/vanilla/dex-config-map.yaml
kubectl rollout restart deployment dex -n auth
```

Note: if need to update the default email address, change the params file located at `manifests\common\user-namespace\base\params.env` before installing Kubeflow.


## Entra ID Configuration
