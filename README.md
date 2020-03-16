## Jenkins and Vault / AppRole Multifactor Authentication for Applications
​
### Secure Introduction
​
AppRole is a secure introduction method to establish machine identity. Whereas, humans may use a LDAP, GitHub or a username and password, we recommend AppRole for machines or apps.
​
In the AppRole workflow, our application gets a Vault token using a Role ID (which is static, and associated with a policy) and a Secret ID (which is dynamic, one time use, and can only be requested by a previously authenticated user or system).
​
With Jenkins as our trusted orchestrator, the Role ID can be introduced in two ways.
- We can store the Role ID in Jenkins
- We can store the Role ID in the Jenkinsfile for each proect
​
#### Enable KV Secrets Engine - Version 1
Let's enable `kv` secrets Version 1 and write a secret for our test application to access.
```
vault secrets enable -version=1 -path=kv1 kv
```
​
```
vault write kv1/javaapps/foo value=bar
```
​
#### Enable AppRole auth method
​
Let's enable the `approle` auth method:
```
vault auth enable -path=approle approle
```
​
When we enable the `approle` auth method, it gets mounted at the `/auth/approle` path.
​
#### Create a policy for Jenkins
​
Let's create a policy called `jenkins` that will be able to retrieve AppRole IDs.
​
```
vault policy write jenkins -<<EOF
# Login with AppRole
path "auth/approle/*" {
  capabilities = [ "create", "read", "update" ]
}
EOF
```
​
#### Application Policy
Let's create a policy for our application to be able to access the secrets in `kv1/javaapps/*`. Let's also give it access to some paths in the KV Version 2 secrets engine we defined earlier (at the `secrets/ path).
​
```
vault policy write java-example -<<EOF
​
path "kv1/javaapps/*" {
  capabilities = ["read", "list", "update"]
}
​
path "secret/data/javaapps/*" {
  capabilities = ["read", "list", "update"]
}
​
EOF
```
​
Let's create a role called `java-example` for our application and associate it with the `java-example` policy.
```
vault write auth/approle/role/java-example \
  secret_id_ttl=60m \
  token_ttl=60m \
  token_max_tll=120m \
  policies="java-example"
```
​
The result is:
```
$ vault read auth/approle/role/java-example
WARNING! The following warnings were returned from Vault:
​
  * The "bound_cidr_list" parameter is deprecated and will be removed in favor
  of "secret_id_bound_cidrs".
​
Key                      Value
---                      -----
bind_secret_id           true
bound_cidr_list          <nil>
local_secret_ids         false
period                   0s
policies                 [java-example]
secret_id_bound_cidrs    <nil>
secret_id_num_uses       0
secret_id_ttl            1h
token_bound_cidrs        <nil>
token_max_ttl            0s
token_num_uses           0
token_ttl                1h
token_type               default
```
​
Let's generate a role id for our application that can be used in conjunction with the Jenkins Token ID to generate a Secret ID that will be provided to the application to access the secret at the paths indicated in the policy.
​
```
vault read auth/approle/role/java-example/role-id
```
​
For example:
```
$ vault read auth/approle/role/java-example/role-id
Key        Value
---        -----
role_id    e631290f-be4b-9836-e5cb-9dab59080e1b
```
​
Let's generate a token for Jenkins that is associated with the `jenkins` policy.
​
Note that Jenkins will not be able to access the secrets.
​
```
vault token create -policy=jenkins
```
​
For example:
```
$ vault token create -policy=jenkins
Key                  Value
---                  -----
token                s.TqeDSLil6xRYpn1yp92hwS7C
token_accessor       eTmu2fzakf66CpNpXJl28OdG
token_duration       768h
token_renewable      true
token_policies       ["default" "jenkins"]
identity_policies    []
policies             ["default" "jenkins"]
```
​
#### Jenkins Setup
​
Let's create Credential in Jenkins for the Jenkins Vault Token and the Role ID for our application.
​
- Credentials -> global -> Add Credential
  - Kind: Secret text
  - Scope: Global
  - Secret: <Jenkins Vault Token>
  - ID: `JENKINS_VAULT_TOKEN`
  - Description: `JENKINS_VAULT_TOKEN`
​
- Credentials -> global -> Add Credential
  - Kind: Secret text
  - Scope: Global
  - Secret: <Java Example Role Id>
  - ID: `JAVA_EXAMPLE_ROLE_ID`
  - Description: `JAVA_EXAMPLE_ROLE_ID`
​
Let's create a new Pipeline with the following definition. You may also need to define the appropriate environment variables to disable the proxy. Let's also double check the vault for `VAULT_ADDR` for your environment.
```
pipeline {
  agent any
stages {  
  stage('Integration Tests') {
      steps {
        sh 'curl -s -o vault.zip https://releases.hashicorp.com/vault/1.3.0/vault_1.3.0_linux_amd64.zip ; yes | unzip vault.zip '
        withCredentials([string(credentialsId: 'JAVA_EXAMPLE_ROLE_ID', variable: 'ROLE_ID'),string(credentialsId: 'JENKINS_VAULT_TOKEN', variable: 'VAULT_TOKEN')]) {
        sh '''
  set -x
   export VAULT_ADDR=http://<vault_endpoint>:8200
  export SECRET_ID=$(./vault write -field=secret_id -f auth/approle/role/java-example/secret-id)
  export VAULT_TOKEN=$(./vault write -field=token auth/approle/login role_id=${ROLE_ID}   secret_id=${SECRET_ID})
    ./vault read kv1/javaapps/foo 
        '''
    }
   }
  }
 }
}
​
```
​
Notte that in our example pipeline, Jenkins:
- retrieves the Secret ID using its Vault Token, which is stored as `JENKINS_VAULT_TOKEN` and the application Role ID, which is stored in `JAVA_EXAMPLE_ROLE_ID`.
- uses the Role ID and the retrieved Secret ID to request a token for the application.
- uses that application token to retrieve the secret at `kv1/javaapps/foo`.
​
Let's run the pipeline. Our output will be similar to:
```
Started by user admin
Running in Durability level: MAX_SURVIVABILITY
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /Users/khemani/.jenkins/workspace/Example App
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Integration Tests)
[Pipeline] sh
+ curl -s -o vault.zip https://releases.hashicorp.com/vault/1.1.0/vault_1.1.0_darwin_amd64.zip
+ yes
+ unzip vault.zip
Archive:  vault.zip
replace vault? [y]es, [n]o, [A]ll, [N]one, [r]ename:   inflating: vault                   
[Pipeline] withCredentials
Masking only exact matches of $ROLE_ID or $VAULT_TOKEN
[Pipeline] {
[Pipeline] sh
+ set -x
+ export VAULT_ADDR=http://127.0.0.1:8200
+ VAULT_ADDR=http://127.0.0.1:8200
++ ./vault write -field=secret_id -f auth/approle/role/java-example/secret-id
+ export SECRET_ID=c5900106-6665-ac3c-68a5-993d13c82753
+ SECRET_ID=c5900106-6665-ac3c-68a5-993d13c82753
++ ./vault write -field=token auth/approle/login role_id=**** secret_id=c5900106-6665-ac3c-68a5-993d13c82753
+ export VAULT_TOKEN=s.rfmCZUmJIbBqVdNIeUERzKlA
+ VAULT_TOKEN=s.rfmCZUmJIbBqVdNIeUERzKlA
+ VAULT_ADDR=http://127.0.0.1:8200
+ ./vault read kv1/javaapps/foo
Key                 Value
---                 -----
refresh_interval    768h
value               bar
[Pipeline] }
[Pipeline] // withCredentials
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
​
```
​
In our example, we executed the last step in a shell script in the pipeline. You would instead provide that token to a Java application.
​
---
​
## Additional App Role Setup
​
#### Let's create some secrets
​
```
vault write kv1/app-1/api-key value='B1qBSifWiCxQm44Id8LY6t'
​
vault write kv1/app-2/foo value='bar'
​
vault write kv1/shared/moo value='b6gV38vbnE2HN'
```
​
---
​
#### App 1 Policy
```
vault policy write app-1 -<<EOF
​
path "kv1/app-1/*" {
  capabilities = ["read", "list", "update"]
}
​
path "secret/data/app-1/*" {
  capabilities = ["read", "list", "update"]
}
​
path "kv1/shared/*" {
  capabilities = ["read", "list"]
}
​
EOF
```
​
#### App-1 Role ID
```
vault write auth/approle/role/app-1 \
  secret_id_ttl=60m \
  token_ttl=60m \
  token_max_tll=120m \
  policies="app-1"
```
​
#### Get App-1 AppRole ID
```
vault read auth/approle/role/app-1/role-id
```
---
#### App 2 Policy
```
vault policy write app-2 -<<EOF
​
path "kv1/app-2/*" {
  capabilities = ["read", "list", "update"]
}
​
path "secret/data/app-2/*" {
  capabilities = ["read", "list", "update"]
}
​
path "kv1/shared/*" {
  capabilities = ["read", "list"]
}
EOF
```
​
#### App-2 Role ID
```
vault write auth/approle/role/app-2 \
  secret_id_ttl=60m \
  token_ttl=60m \
  token_max_tll=120m \
  policies="app-2"
```
​
#### Get App-2 AppRole ID
```
vault read auth/approle/role/app-2/role-id
```
---
​
#### Jenkin-1
```
vault policy write jenkins-1 -<<EOF
# Login with AppRole
path "auth/approle/role/app-1/*" {
  capabilities = [ "create", "read", "update" ]
}
EOF
```
​
```
$ vault token create -policy=jenkins-1
Key                  Value
---                  -----
token                s.lnHFLXj090cYc0CzS7mmc2Fi
token_accessor       NMCmpV1C3uM9ufE2vbXuDeBp
token_duration       768h
token_renewable      true
token_policies       ["default" "jenkins-1"]
identity_policies    []
policies             ["default" "jenkins-1"]
```
​
---
​
#### Jenkin-2
​
```
vault policy write jenkins-2 -<<EOF
# Login with AppRole
path "auth/approle/role/app-2/*" {
  capabilities = [ "create", "read", "update" ]
}
EOF
```
​
​
```
$ vault token create -policy=jenkins-2
Key                  Value
---                  -----
token                s.AoxhOgi3AyJ86TPUkIhd6NB1
token_accessor       2049ifhODnNERGOYLNn6tPcq
token_duration       768h
token_renewable      true
token_policies       ["default" "jenkins-2"]
identity_policies    []
policies             ["default" "jenkins-2"]
```
