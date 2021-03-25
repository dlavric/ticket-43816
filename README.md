# ticket-43816
Vault with Oracle DB secrets engine and a static role + password policy + rotation 

Vault 1.7.0
Oracle Plugin 0.4.0

Latest version at 25th March 2021



```shell
$ vagrant ssh vault
```

Connect to this databasE:
```shell
$ source /vagrant/sw/instantclient.env 

$ sqlplus system/password@//db.test:1521/XEPDB1
```

```sql
create user myuser identified by password;

grant connect to myuser;

grant all privileges to myuser;
```
!! DO NOT create db ROLE in Oracle DB!!

```shell
$ export VAULT_ADDR="http://127.0.0.1:8200"
```

- create a policy
```shell
$ tee admin-policy.hcl <<EOF
# Mount secrets engines
path "sys/mounts/*" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
}

# Configure the database secrets engine and create roles
path "database/*" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
}

# Write ACL policies
path "sys/policies/acl/*" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
}

# Manage tokens for verification
path "auth/token/create" {
  capabilities = [ "create", "read", "update", "delete", "list", "sudo" ]
}
EOF
```

```shell
$ vault policy write admin admin-policy.hcl
```

- Create a password policy:
```shell
$ tee password-policy.hcl <<EOF
length = 12
rule "charset" {
charset = "abcdefghijklmnopqrstuvwxyz"
min-chars = 5
}
rule "charset" {
charset = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
min-chars = 1
}
rule "charset" {
charset = "0123456789"
min-chars = 1
}
rule "charset" {
charset = "!@#$%^&*"
min-chars = 2
}
EOF
```

```shell
$ vault write sys/policies/password/oracle-database-plugin-policy policy=@password-policy.hcl
```
Output
```
Success! Data written to: sys/policies/password/oracle-database-plugin
```


2. Enable the database secrets engine

is done

3. Configure the secrets engine

is done



- Create the rotation.sql file:
```shell
tee rotation.sql <<EOF
ALTER USER {{name}} IDENTIFIED BY "{{password}}";   
EOF
```

```shell
vault write database/config/my-oracle-database \
  plugin_name="oracle-database-plugin" \
  connection_url="system/password@db.test:1521/XEPDB1" \
  allowed_roles="my-role,my-user" \
  password_policy="oracle-database-plugin-policy" \
  username="myuser"
```

```shell
$ tee apps.hcl <<EOF
# Get credentials from the database secrets engine
path "database/static-roles/my-user" {
  capabilities = [ "read" ]
}
EOF
```

```shell
$ vault policy write apps apps.hcl
```

```shell
$ vault token create -policy="apps"
```
Output:
```
Key                  Value
---                  -----
token                s.9nTdEPhTuUspi6sE4jDtjsxZ
token_accessor       1MNzuEAn3Uk69RmjQpRo2DNM
token_duration       168h
token_renewable      true
token_policies       ["apps" "default"]
identity_policies    []
policies             ["apps" "default"]
```

```shell
$ vault write database/static-roles/my-user \
    db_name=my-oracle-database \
    rotation_statements=@rotation.sql \
    username="myuser" \
    rotation_period=6
```
Output:
```
Success! Data written to: database/static-roles/my-user
```

```shell
$ vault read database/static-creds/my-user
```
Output:
```
Key                    Value
---                    -----
last_vault_rotation    2021-03-25T14:27:43.584674454Z
password               N3*JqyvtHmJ#
rotation_period        6s
ttl                    4s
username               myuser
```

```shell
$ vault read sys/policies/password/oracle-database-plugin-policy/generate
```
Output:
```
Key         Value
---         -----
password    YcXviV&*ecJ5
```

