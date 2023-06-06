
## 볼트 워크샵


```
# 볼트 버전 확인
Vault v1.8.3 (73e85c3c21dfd1e835ded0053f08e3bd73a24ad6)

# 볼트 CLI 명령 목록 보기
Usage: vault <command> [args]

Common commands:
    read        Read data and retrieves secrets
    write       Write data, configuration, and secrets
    delete      Delete secrets and configuration
    list        List data or secrets
    login       Authenticate locally
    agent       Start a Vault agent
    server      Start a Vault server
    status      Print seal and HA status
    unwrap      Unwrap a wrapped secret

Other commands:
    audit          Interact with audit devices
    auth           Interact with auth methods
    debug          Runs the debug command
    kv             Interact with Vault's Key-Value storage
    lease          Interact with leases
    monitor        Stream log messages from a Vault server
    namespace      Interact with namespaces
    operator       Perform operator-specific tasks
    path-help      Retrieve API help for paths
    plugin         Interact with Vault plugins and catalog
    policy         Interact with policies
    print          Prints runtime configurations
    secrets        Interact with secrets engines
    ssh            Initiate an SSH session
    token          Interact with tokens
```


### 볼트 dev 모드 실행, 시크릿 작성

Vault server는 "dev" 또는 "production" 모드로 동작할 수 있다.

**Vault를 Dev Server mode로 기동**

```
vault server -dev -dev-listen-address=0.0.0.0:8200 -dev-root-token-id=root
```

Vault UI 접속

![image](https://github.com/jiwonYun9332/infra/blob/ddd16d12ed3859ce4b655ed69754c618f963da83/HashiCorp/vault/vaultImage0601/1.jpg)

**KV v2 시크릿 엔진 생성**

my-first-secret 을 path로, "age"를 시크릿의 첫 번째 키로 시크릿을 생성한다.

```
vault kv put secret/my-first-secret age=24
Key              Value
---              -----
created_time     2023-06-06T14:08:26.94749268Z
deletion_time    n/a
destroyed        false
version          1
```

다시 UI로 접속해보면 my-first-secret 가 생긴 것을 볼 수 있다.

![image](https://github.com/jiwonYun9332/infra/blob/58d6ea4404035a8238cf5dcce838fbeaf25a9196/HashiCorp/vault/vaultImage0601/2.jpg)

Create new version으로 key를 추가하거나 key 값을 변경할 수 있으며, 변경된 내용은 버전으로 관리된다.

![image](https://github.com/jiwonYun9332/infra/blob/58d6ea4404035a8238cf5dcce838fbeaf25a9196/HashiCorp/vault/vaultImage0601/3.jpg)


### Vault HTTP API

```
curl http://localhost:8200/v1/sys/health | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   294  100   2{0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
94   "initialized": true ,
    "sealed": false,
  "standby": false,
  "performance_standby": false,
  "replication_performance_mode": "disabled",
0   "replication_dr_mode": "disabled",
      "server_time_utc": 16860611640,
    "version": "1.8.3",
 21  "cluster_name": "vault-cluster-117d717e",
0k    "cluster_id": "528dfcf3-2b3e-7c87-37de-f356d7a62462"
   } 
0 --:--:-- --:--:-- --:--:--  287k

**생성한 my-first-secret 시크릿을 통해 API 요청**

```
curl --header "X-Vault-Token: root" http://localhost:8200/v1/secret/da
ta/my-first-secret | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   288  100   288    0     0  62418      0 --:--:-- --:--:-- --:--:-- 72000
{
  "request_id": "19de1791-bfc3-6429-830e-bd1861cca07e",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "data": {
      "age": "20"
    },
    "metadata": {
      "created_time": "2023-06-06T14:14:34.88672702Z",
      "deletion_time": "",
      "destroyed": false,
      "version": 4
    }
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null
}
```

### Vault server를 Production 모드로 기동

프로덕션으로 서버 실행 시 구성 파일에서 해당 구성을 가져온다.

vault-config.hcl 구성 파일을 확인한다.

```
listener "tcp" {
 address = "0.0.0.0:8200"
 tls_disable = 1
}
storage "file" {
  path = "/vault/file"
}
disable_mlock = true
api_addr = "http://localhost:8200"
ui=true
```

볼트 서버 프로덕션 모드로 실행

```
vault server -config=/vault/config/vault-config.hcl
```

볼트는 운영모드로 실행하면 잠겨있으며 초기화가 필요하다.

볼트 초기화

```
vault operator init -key-shares=1 -key-threshold=1
Unseal Key 1: 952srHPajsXyQh3fJIiUz//SeZ3a+AFvjO4pNo/d15s=

Initial Root Token: s.7S4xbEh0PUzpadvExgBiA5Ha

Vault initialized with 1 key shares and a key threshold of 1. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 1 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 1 keys to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

토큰 값 환경변수 설정

```
export VAULT_TOKEN=s.7S4xbEh0PUzpadvExgBiA5Ha
echo "export VAULT_TOKEN=$VAULT_TOKEN" >> /root/.profile
```

vault status로 확인 시 초기화는 되었지만 Sealed 값이 true으로 잠겨있는 상황이다.

```
vault status
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       1
Threshold          1
Unseal Progress    0/1
Unseal Nonce       n/a
Version            1.8.3
Storage Type       file
HA Enabled         false
```

볼트 잠금 해제

```
vault operator unseal
Unseal Key (will be hidden): 
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.8.3
Storage Type    file
Cluster Name    vault-cluster-3774f1e6
Cluster ID      5ff6e6b2-b546-d743-073f-585dfc606c81
HA Enabled      false
```

Sealed가 잠금 해제되면서 false 된 것을 확인할 수 있다. 잠금 해제 작업은 볼트 서버가 다시 실행될 때마다
해야한다.

주의할 점으로 루트 토큰과 잠금 해제 키를 저장하는 것을 잊지 말아야 한다.

### Productio Vault 서버에 KV v2 시크릿 엔진 활성화

KV v2 시크릿 엔진을 기본 경로로 마운트

```
vault secrets enable -version=2 kv
```

새로 생성된 시크릿엔진에 시크릿 만들기

```
vault kv put kv/a-secret value=1234
```

### 사용자 인증

사용자 이름/비밀번호 인증 방법을 활성화

```
vault auth enable userpass
Success! Enabled userpass auth method at: userpass/
```

```
vault write auth/userpass/users/user1 password=user1
Success! Data written to: auth/userpass/users/user1
```

Vault CLI를 사용하여 로그인

```
vault login -method=userpass username=user1 password=user1
WARNING! The VAULT_TOKEN environment variable is set! This takes precedence
over the value set by this command. To use the value set by this command,
unset the VAULT_TOKEN environment variable or set it to the token displayed
below.

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                    Value
---                    -----
token                  s.o9bASQ1ucMKovlZdzSmI4AKV
token_accessor         IwgwYPuVJpfVvAnjMGp4CWVk
token_duration         768h
token_renewable        true
token_policies         ["default"]
identity_policies      []
policies               ["default"]
token_meta_username    user1
```

기존 저장한 토큰 값 환경변수 해제

```
unset VAULT_TOKEN
```

현재 토큰에 대한 정보 확인

```
vault token lookup
Key                 Value
---                 -----
accessor            IwgwYPuVJpfVvAnjMGp4CWVk
creation_time       1686071995
creation_ttl        768h
display_name        userpass-user1
entity_id           79ccfe3d-b7b8-d8f9-0e57-4571de47e3f9
expire_time         2023-07-08T17:19:55.523692477Z
explicit_max_ttl    0s
id                  s.o9bASQ1ucMKovlZdzSmI4AKV
issue_time          2023-06-06T17:19:55.523699272Z
meta                map[username:user1]
num_uses            0
orphan              true
path                auth/userpass/login/user1
policies            [default]
renewable           true
ttl                 767h58m1s
type                service
```

KV v2 시크릿 엔진의 시크릿을 읽기

```
vault kv get kv/a-secret
Error making API request.

URL: GET http://localhost:8200/v1/sys/internal/ui/mounts/kv/a-secret
Code: 403. Errors:

* preflight capability check returned 403, please ensure client's policies grant access to path "kv/a-secret/"
```

user1 권한으로는 해당 시크릿을 읽을 수 없다.

### Vault 내의 서로 다른 시크릿에 접근할 수 있도록 두 명의 사용자을 위한 정책 정의


user-1-policy.hci

```
path "kv/data/user1/*" {
  capabilities = ["create", "update", "read", "delete"]
}
path "kv/delete/user1/*" {
  capabilities = ["update"]
}
path "kv/metadata/user1/*" {
  capabilities = ["list", "read", "delete"]
}
path "kv/destroy/user1/*" {
  capabilities = ["update"]
}

# Additional access for UI
path "kv/metadata" {
  capabilities = ["list"]
}
```

user-1-policy.hci

```
path "kv/data/user2/*" {
  capabilities = ["create", "update", "read", "delete"]
}
path "kv/delete/user2/*" {
  capabilities = ["update"]
}
path "kv/metadata/user2/*" {
  capabilities = ["list", "read", "delete"]
}
path "kv/destroy/user2/*" {
  capabilities = ["update"]
}

# Additional access for UI
path "kv/metadata" {
  capabilities = ["list"]
}
```

user2 계정 생성

```
vault write auth/userpass/users/user2 password=user2
Success! Data written to: auth/userpass/users/user2
```

정책 생성

```
vault policy write user_1 /vault/policies/user-1-policy.hcl
Success! Uploaded policy: user_1
vault policy write user2 /vault/policies/user-2-policy.hcl
Success! Uploaded policy: user2
```

정책 할당

```
vault write auth/userpass/users/user1/policies policies=user1
Success! Data written to: auth/userpass/users/user1/policies
vault write auth/userpass/users/user2/policies policies=user2
Success! Data written to: auth/userpass/users/user2/policies
```

user1 계정이 읽을 수 있는 시크릿 생성

![image](https://github.com/jiwonYun9332/infra/blob/ff770cdb45d742cd40234652ed1b5ff4c467c4b9/HashiCorp/vault/vaultImage0601/4.jpg)

user1 계정의 시크릿 age kv 읽기

```
vault kv get kv/user1/age
====== Metadata ======
Key              Value
---              -----
created_time     2023-06-06T17:37:55.022471414Z
deletion_time    n/a
destroyed        false
version          1

=== Data ===
Key    Value
---    -----
age    24
```

user1 계정의 시크릿 중 weight kv 쓰기

```
vault kv put kv/user1/weight weight=150
Key              Value
---              -----
created_time     2023-06-06T17:44:14.460182501Z
deletion_time    n/a
destroyed        false
version          1
vault-server:~# time
BusyBox v1.33.1 () multi-call binary.
```
