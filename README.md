# Keycloak Load Testing

This repository presents the organization and results of a load test against Keycloak.

## System Under Test

### Versions

| Component   | Version |
|-------------|---------|
| Red Hat SSO |     7.4 |
| PostgreSQL  |      10 |
| MariaDB     |    10.3 |
| Traefik     |     2.3 |
| OpenLDAP    |     2.4 |

### System preparation

* Install CentOS Stream 8 or RHEL8 on the target server.
* Do not forget to register and attach subscriptions when using RHEL.
* Do a `sudo podman login registry.redhat.io` if you plan to use the Red Hat branded images

## Sizes

| T-shirt size | Realms | Clients |  Users | Concurrent sessions |
|--------------|--------|---------|--------|---------------------|
| S            |      1 |      10 |    100 |                1000 |
| M            |      5 |     100 |   1000 |                1000 |
| L            |     10 |    1000 |  10000 |                   - |

Details:

* Realms: total number of realms in the Keycloak database
* Clients: number of clients **per realm**
* Users: number of users **per realm**
* Concurrent sessions: total number of sessions **across all realms**

Note: the concurrent session count is used for the refresh-token, tokeninfo and userinfo tests.

## Setup

Edit the Ansible inventory and change the connection details.

```sh
cp setup/inventory.sample setup/inventory
vi setup/inventory
```

On the machine running K6, set the URL of the Keycloak installation (should match the hostname in your inventory file).

```sh
export KEYCLOAK_URL=http://your.keycloak.hostname/auth
```

Install [kci](https://github.com/nmasse-itix/keycloak-import-realm/releases) somewhere in your PATH and configure it to target your Keycloak instance.

```sh
kci config set realm --value master
kci config set login --value admin
kci config set password --value admin
kci config set keycloak_url --value http://your.keycloak.hostname/auth
```

Install [k6](https://k6.io/docs/getting-started/installation).

## Scenario "BASELINE"

### Preparation

```sh
ansible-playbook -i setup/inventory setup/provision.yaml -e @setup/scenarios/baseline.yaml
rm -f k6/data/*.json
kci generate --realms 5 --clients 100 --users 1000 --template realm-templates/baseline.template --target k6/data/
kci import k6/data/*.json
```

PostgreSQL database size:

```
# du -sh /srv/postgresql/data/
109M    /srv/postgresql/data/
```

### Test: Login

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
REALM_COUNT=5 VU_COUNT=12 k6 run -o datadog k6/login.js
```

**Results**

```
checks.....................: 100.00% ✓ 35916 ✗ 0    
data_received..............: 122 MB  185 kB/s
data_sent..................: 33 MB   51 kB/s
http_req_blocked...........: avg=315.48µs min=1.34µs  med=8.26µs   max=35.3ms  p(90)=1.02ms   p(95)=1.09ms  
http_req_connecting........: avg=274.58µs min=0s      med=0s       max=16.94ms p(90)=884.84µs p(95)=959.38µs
http_req_duration..........: avg=209.9ms  min=2.34ms  med=44.84ms  max=1.6s    p(90)=768.44ms p(95)=922.72ms
http_req_receiving.........: avg=201.79µs min=19.73µs med=185.15µs max=5.99ms  p(90)=319.33µs p(95)=352.43µs
http_req_sending...........: avg=72.26µs  min=10.05µs med=63.44µs  max=1.91ms  p(90)=156.74µs p(95)=184.85µs
http_req_tls_handshaking...: avg=0s       min=0s      med=0s       max=0s      p(90)=0s       p(95)=0s      
http_req_waiting...........: avg=209.63ms min=2.01ms  med=44.61ms  max=1.6s    p(90)=768.06ms p(95)=922.39ms
http_reqs..................: 35916   54.407101/s
iteration_duration.........: avg=632.84ms min=105ms   med=617.38ms max=1.63s   p(90)=1.04s    p(95)=1.11s   
iterations.................: 11972   18.1357/s
script_errors..............: 0.00%   ✓ 0     ✗ 11972
vus........................: 1       min=1   max=12 
vus_max....................: 12      min=12  max=12 
```

### Test: Userinfo

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
SESSION_COUNT=1000 REALM_COUNT=5 VU_COUNT=20 k6 run -o datadog k6/userinfo.js
```

**Results**

```
checks.....................: 100.00% ✓ 2632361 ✗ 0      
data_received..............: 888 MB  1.2 MB/s
data_sent..................: 3.4 GB  4.5 MB/s
http_req_blocked...........: avg=3.16µs  min=965ns  med=2.33µs  max=84.49ms  p(90)=5.09µs  p(95)=6.19µs 
http_req_connecting........: avg=6ns     min=0s     med=0s      max=1.22ms   p(90)=0s      p(95)=0s     
http_req_duration..........: avg=4.63ms  min=1.27ms med=3.55ms  max=175.48ms p(90)=7.88ms  p(95)=10.08ms
http_req_receiving.........: avg=39.01µs min=9.24µs med=31.62µs max=21.85ms  p(90)=63.71µs p(95)=76.52µs
http_req_sending...........: avg=19.1µs  min=5.78µs med=15.28µs max=17.7ms   p(90)=32.44µs p(95)=38.93µs
http_req_tls_handshaking...: avg=0s      min=0s     med=0s      max=0s       p(90)=0s      p(95)=0s     
http_req_waiting...........: avg=4.57ms  min=1.24ms med=3.49ms  max=175.18ms p(90)=7.82ms  p(95)=10.02ms
http_reqs..................: 2633361 3458.775176/s
iteration_duration.........: avg=4.82ms  min=1.37ms med=3.74ms  max=1m40s    p(90)=8.06ms  p(95)=10.25ms
iterations.................: 2632361 3457.461731/s
script_errors..............: 0.00%   ✓ 0       ✗ 2632361
vus........................: 1       min=0     max=20   
vus_max....................: 20      min=20    max=20   
```

### Test: Refresh Token

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
SESSION_COUNT=1000 REALM_COUNT=5 VU_COUNT=10 k6 run -o datadog k6/refresh-token.js
```

**Results**

```
checks.....................: 100.00% ✓ 305174 ✗ 0     
data_received..............: 987 MB  1.3 MB/s
data_sent..................: 292 MB  383 kB/s
http_req_blocked...........: avg=7.16µs   min=1.14µs  med=7.24µs   max=3.4ms    p(90)=10.57µs  p(95)=11.86µs
http_req_connecting........: avg=42ns     min=0s      med=0s       max=3.33ms   p(90)=0s       p(95)=0s     
http_req_duration..........: avg=24.36ms  min=5.99ms  med=23.69ms  max=182.84ms p(90)=30ms     p(95)=35.55ms
http_req_receiving.........: avg=159.74µs min=13.63µs med=135.74µs max=24.01ms  p(90)=232.59µs p(95)=287.2µs
http_req_sending...........: avg=47.17µs  min=7.82µs  med=52.17µs  max=7.75ms   p(90)=72.48µs  p(95)=79.28µs
http_req_tls_handshaking...: avg=0s       min=0s      med=0s       max=0s       p(90)=0s       p(95)=0s     
http_req_waiting...........: avg=24.15ms  min=5.85ms  med=23.48ms  max=182.57ms p(90)=29.8ms   p(95)=35.35ms
http_reqs..................: 306174  401.817684/s
iteration_duration.........: avg=25.13ms  min=6.23ms  med=24.4ms   max=1m41s    p(90)=30.56ms  p(95)=35.66ms
iterations.................: 305174  400.505301/s
script_errors..............: 0.00%   ✓ 0      ✗ 305174
vus........................: 1       min=0    max=12  
vus_max....................: 12      min=12   max=12  
```

### Test: Tokeninfo

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
SESSION_COUNT=1000 REALM_COUNT=5 VU_COUNT=25 k6 run -o datadog k6/tokeninfo.js
```

**Results**

```
checks.....................: 100.00% ✓ 2137119 ✗ 0      
data_received..............: 1.5 GB  1.9 MB/s
data_sent..................: 3.1 GB  4.1 MB/s
http_req_blocked...........: avg=3.8µs   min=1.07µs med=2.86µs  max=8.76ms   p(90)=6.22µs  p(95)=7.34µs 
http_req_connecting........: avg=10ns    min=0s     med=0s      max=1.38ms   p(90)=0s      p(95)=0s     
http_req_duration..........: avg=7.1ms   min=1.47ms med=6.41ms  max=174.58ms p(90)=11.17ms p(95)=13.13ms
http_req_receiving.........: avg=45.35µs min=9.96µs med=36.06µs max=28.57ms  p(90)=73.15µs p(95)=87.4µs 
http_req_sending...........: avg=26.13µs min=7.74µs med=20.73µs max=14.53ms  p(90)=44µs    p(95)=51.74µs
http_req_tls_handshaking...: avg=0s      min=0s     med=0s      max=0s       p(90)=0s      p(95)=0s     
http_req_waiting...........: avg=7.03ms  min=1.43ms med=6.34ms  max=174.31ms p(90)=11.09ms p(95)=13.06ms
http_reqs..................: 2138119 2811.929231/s
iteration_duration.........: avg=7.41ms  min=1.6ms  med=6.72ms  max=1m39s    p(90)=11.46ms p(95)=13.43ms
iterations.................: 2137119 2810.614089/s
script_errors..............: 0.00%   ✓ 0       ✗ 2137119
vus........................: 1       min=0     max=25   
vus_max....................: 25      min=25    max=25   
```

## Scenario "Offline Tokens"

### Preparation

```sh
ansible-playbook -i setup/inventory setup/provision.yaml -e @setup/scenarios/baseline.yaml
rm -f k6/data/*.json
kci generate --realms 5 --clients 100 --users 1000 --template realm-templates/baseline.template --target k6/data/
kci import k6/data/*.json
```

### Test: Login

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
KEYCLOAK_OFFLINE_TOKENS=true REALM_COUNT=5 VU_COUNT=12 k6 run -o datadog k6/login.js
```

**Results**

```
checks.....................: 100.00% ✓ 32454 ✗ 0    
data_received..............: 112 MB  169 kB/s
data_sent..................: 31 MB   46 kB/s
http_req_blocked...........: avg=358.18µs min=1.47µs   med=9.51µs   max=15.1ms  p(90)=1.07ms   p(95)=1.15ms  
http_req_connecting........: avg=301.97µs min=0s       med=0s       max=14.93ms p(90)=910.12µs p(95)=984.9µs 
http_req_duration..........: avg=232.04ms min=2.53ms   med=89.36ms  max=1.45s   p(90)=709.37ms p(95)=843.96ms
http_req_receiving.........: avg=239.9µs  min=29.26µs  med=216.48µs max=6.44ms  p(90)=361.2µs  p(95)=398.6µs 
http_req_sending...........: avg=101.21µs min=11.27µs  med=79.6µs   max=2.14ms  p(90)=182.41µs p(95)=196.37µs
http_req_tls_handshaking...: avg=0s       min=0s       med=0s       max=0s      p(90)=0s       p(95)=0s      
http_req_waiting...........: avg=231.7ms  min=2.1ms    med=89.11ms  max=1.45s   p(90)=708.95ms p(95)=843.55ms
http_reqs..................: 32454   49.158051/s
iteration_duration.........: avg=700.33ms min=105.78ms med=701.15ms max=1.57s   p(90)=1.01s    p(95)=1.09s   
iterations.................: 10818   16.386017/s
script_errors..............: 0.00%   ✓ 0     ✗ 10818
vus........................: 1       min=1   max=12 
vus_max....................: 12      min=12  max=12 
```

### Test: Userinfo

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
SESSION_COUNT=1000 KEYCLOAK_OFFLINE_TOKENS=true REALM_COUNT=5 VU_COUNT=20 k6 run -o datadog k6/userinfo.js
```

**Results**

```
checks.....................: 100.00% ✓ 2398658 ✗ 0      
data_received..............: 810 MB  1.1 MB/s
data_sent..................: 3.3 GB  4.3 MB/s
http_req_blocked...........: avg=3.56µs  min=973ns  med=2.63µs  max=10.86ms  p(90)=6.01µs  p(95)=7.18µs 
http_req_connecting........: avg=7ns     min=0s     med=0s      max=1.58ms   p(90)=0s      p(95)=0s     
http_req_duration..........: avg=5.07ms  min=1.24ms med=3.81ms  max=182.25ms p(90)=8.87ms  p(95)=11.88ms
http_req_receiving.........: avg=44.27µs min=9.18µs med=34.77µs max=20.73ms  p(90)=74.63µs p(95)=88.38µs
http_req_sending...........: avg=21.84µs min=6.1µs  med=16.98µs max=22.24ms  p(90)=38.12µs p(95)=45.3µs 
http_req_tls_handshaking...: avg=0s      min=0s     med=0s      max=0s       p(90)=0s      p(95)=0s     
http_req_waiting...........: avg=5.01ms  min=1.21ms med=3.74ms  max=182.05ms p(90)=8.81ms  p(95)=11.81ms
http_reqs..................: 2399658 3129.409441/s
iteration_duration.........: avg=5.29ms  min=1.34ms med=4.03ms  max=1m46s    p(90)=9.08ms  p(95)=12.06ms
iterations.................: 2398658 3128.105334/s
script_errors..............: 0.00%   ✓ 0       ✗ 2398658
vus........................: 1       min=0     max=20   
vus_max....................: 20      min=20    max=20   
```

### Test: Refresh Token

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
SESSION_COUNT=1000 KEYCLOAK_OFFLINE_TOKENS=true REALM_COUNT=5 VU_COUNT=10 k6 run -o datadog k6/refresh-token.js
```

**Results**

```
checks.....................: 100.00% ✓ 289096 ✗ 0     
data_received..............: 961 MB  1.3 MB/s
data_sent..................: 276 MB  359 kB/s
http_req_blocked...........: avg=7.16µs   min=1.15µs  med=7.17µs   max=2.9ms    p(90)=10.65µs  p(95)=11.88µs 
http_req_connecting........: avg=29ns     min=0s      med=0s       max=928.55µs p(90)=0s       p(95)=0s      
http_req_duration..........: avg=21.45ms  min=6.1ms   med=19.91ms  max=272.65ms p(90)=31.33ms  p(95)=35.26ms 
http_req_receiving.........: avg=161.68µs min=12.82µs med=148.06µs max=14.59ms  p(90)=232.79µs p(95)=275.84µs
http_req_sending...........: avg=47.88µs  min=8.16µs  med=52.56µs  max=8.9ms    p(90)=73.96µs  p(95)=81.44µs 
http_req_tls_handshaking...: avg=0s       min=0s      med=0s       max=0s       p(90)=0s       p(95)=0s      
http_req_waiting...........: avg=21.24ms  min=5.94ms  med=19.7ms   max=272.39ms p(90)=31.12ms  p(95)=35.05ms 
http_reqs..................: 290096  377.516374/s
iteration_duration.........: avg=22.2ms   min=6.34ms  med=20.57ms  max=1m48s    p(90)=31.77ms  p(95)=35.58ms 
iterations.................: 289096  376.215024/s
script_errors..............: 0.00%   ✓ 0      ✗ 289096
vus........................: 1       min=0    max=10  
vus_max....................: 10      min=10   max=10  
```

### Test: Tokeninfo

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
SESSION_COUNT=1000 KEYCLOAK_OFFLINE_TOKENS=true REALM_COUNT=5 VU_COUNT=25 k6 run -o datadog k6/tokeninfo.js
```

**Results**

```
checks.....................: 100.00% ✓ 2072898 ✗ 0      
data_received..............: 1.5 GB  2.0 MB/s
data_sent..................: 3.2 GB  4.1 MB/s
http_req_blocked...........: avg=3.95µs  min=1.08µs med=3.12µs  max=12.43ms  p(90)=6.04µs  p(95)=7.19µs 
http_req_connecting........: avg=14ns    min=0s     med=0s      max=7.6ms    p(90)=0s      p(95)=0s     
http_req_duration..........: avg=7.31ms  min=1.55ms med=6.71ms  max=184.68ms p(90)=11.06ms p(95)=13.02ms
http_req_receiving.........: avg=47.27µs min=9.48µs med=39.35µs max=51.04ms  p(90)=73.45µs p(95)=88.26µs
http_req_sending...........: avg=27.34µs min=7.82µs med=22.95µs max=18.27ms  p(90)=43.61µs p(95)=51.82µs
http_req_tls_handshaking...: avg=0s      min=0s     med=0s      max=0s       p(90)=0s      p(95)=0s     
http_req_waiting...........: avg=7.24ms  min=1.47ms med=6.64ms  max=184.42ms p(90)=10.98ms p(95)=12.94ms
http_reqs..................: 2073898 2711.764459/s
iteration_duration.........: avg=7.64ms  min=1.74ms med=7.03ms  max=1m44s    p(90)=11.37ms p(95)=13.33ms
iterations.................: 2072898 2710.456891/s
script_errors..............: 0.00%   ✓ 0       ✗ 2072898
vus........................: 1       min=0     max=25   
vus_max....................: 25      min=25    max=25   
```

## Scenario "MariaDB instead of PostgreSQL"

### Preparation

```sh
ansible-playbook -i setup/inventory setup/provision.yaml -e @setup/scenarios/mariadb.yaml
rm -f k6/data/*.json
kci generate --realms 5 --clients 100 --users 1000 --template realm-templates/baseline.template --target k6/data/
kci import k6/data/*.json
```

### Test: Login

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
REALM_COUNT=5 VU_COUNT=12 k6 run -o datadog k6/login.js
```

**Results**

```
checks.....................: 100.00% ✓ 36480 ✗ 0    
data_received..............: 124 MB  188 kB/s
data_sent..................: 34 MB   51 kB/s
http_req_blocked...........: avg=319.29µs min=1.31µs  med=8.28µs   max=31.38ms p(90)=1.03ms   p(95)=1.09ms  
http_req_connecting........: avg=278.38µs min=0s      med=0s       max=13.53ms p(90)=888.12µs p(95)=957.87µs
http_req_duration..........: avg=206.63ms min=2.46ms  med=44.81ms  max=1.43s   p(90)=731.35ms p(95)=901.01ms
http_req_receiving.........: avg=204.63µs min=25.26µs med=186.38µs max=15.28ms p(90)=325.14µs p(95)=360.27µs
http_req_sending...........: avg=72.28µs  min=10.77µs med=63.4µs   max=1.5ms   p(90)=156.79µs p(95)=184.44µs
http_req_tls_handshaking...: avg=0s       min=0s      med=0s       max=0s      p(90)=0s       p(95)=0s      
http_req_waiting...........: avg=206.35ms min=2.17ms  med=44.56ms  max=1.43s   p(90)=731.03ms p(95)=900.69ms
http_reqs..................: 36480   55.259275/s
iteration_duration.........: avg=623.04ms min=104.6ms med=601.62ms max=1.55s   p(90)=1.02s    p(95)=1.1s    
iterations.................: 12160   18.419758/s
script_errors..............: 0.00%   ✓ 0     ✗ 12160
vus........................: 1       min=1   max=12 
vus_max....................: 12      min=12  max=12 
```

### Test: Userinfo

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
SESSION_COUNT=1000 KEYCLOAK_OFFLINE_TOKENS=true REALM_COUNT=5 VU_COUNT=20 k6 run -o datadog k6/userinfo.js
```

**Results**

```
checks.....................: 100.00% ✓ 2413886 ✗ 0      
data_received..............: 815 MB  1.1 MB/s
data_sent..................: 3.3 GB  4.3 MB/s
http_req_blocked...........: avg=4.51µs  min=973ns  med=3.57µs  max=27.53ms  p(90)=7.14µs  p(95)=8.16µs  
http_req_connecting........: avg=7ns     min=0s     med=0s      max=1.44ms   p(90)=0s      p(95)=0s      
http_req_duration..........: avg=4.98ms  min=1.42ms med=3.95ms  max=189.38ms p(90)=8.3ms   p(95)=10.56ms 
http_req_receiving.........: avg=54.56µs min=9.46µs med=46µs    max=41.11ms  p(90)=87.08µs p(95)=101.83µs
http_req_sending...........: avg=27.22µs min=6.17µs med=22.82µs max=30.54ms  p(90)=45.04µs p(95)=51.97µs 
http_req_tls_handshaking...: avg=0s      min=0s     med=0s      max=0s       p(90)=0s      p(95)=0s      
http_req_waiting...........: avg=4.9ms   min=1.36ms med=3.87ms  max=189.14ms p(90)=8.21ms  p(95)=10.48ms 
http_reqs..................: 2414886 3147.248294/s
iteration_duration.........: avg=5.25ms  min=1.51ms med=4.24ms  max=1m46s    p(90)=8.56ms  p(95)=10.82ms 
iterations.................: 2413886 3145.945024/s
script_errors..............: 0.00%   ✓ 0       ✗ 2413886
vus........................: 1       min=0     max=20   
vus_max....................: 20      min=20    max=20   
```

### Test: Refresh Token

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
SESSION_COUNT=1000 KEYCLOAK_OFFLINE_TOKENS=true REALM_COUNT=5 VU_COUNT=10 k6 run -o datadog k6/refresh-token.js
```

**Results**

```
checks.....................: 100.00% ✓ 300333 ✗ 0     
data_received..............: 999 MB  1.3 MB/s
data_sent..................: 287 MB  376 kB/s
http_req_blocked...........: avg=7.79µs   min=1.16µs  med=7.72µs   max=2.94ms   p(90)=10.92µs  p(95)=12.08µs 
http_req_connecting........: avg=32ns     min=0s      med=0s       max=1.72ms   p(90)=0s       p(95)=0s      
http_req_duration..........: avg=20.55ms  min=6.06ms  med=19.11ms  max=228.69ms p(90)=29.49ms  p(95)=33.14ms 
http_req_receiving.........: avg=165.92µs min=13.85µs med=148.99µs max=18.11ms  p(90)=239.94µs p(95)=285.33µs
http_req_sending...........: avg=52.13µs  min=8.11µs  med=55.07µs  max=9.59ms   p(90)=75.46µs  p(95)=82.97µs 
http_req_tls_handshaking...: avg=0s       min=0s      med=0s       max=0s       p(90)=0s       p(95)=0s      
http_req_waiting...........: avg=20.33ms  min=5.91ms  med=18.89ms  max=228.43ms p(90)=29.26ms  p(95)=32.92ms 
http_reqs..................: 301333  394.417707/s
iteration_duration.........: avg=21.35ms  min=6.27ms  med=19.84ms  max=1m43s    p(90)=30.1ms   p(95)=33.6ms  
iterations.................: 300333  393.108797/s
script_errors..............: 0.00%   ✓ 0      ✗ 300333
vus........................: 1       min=0    max=10  
vus_max....................: 10      min=10   max=10  
```

### Test: Tokeninfo

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
SESSION_COUNT=1000 KEYCLOAK_OFFLINE_TOKENS=true REALM_COUNT=5 VU_COUNT=25 k6 run -o datadog k6/tokeninfo.js
```

**Results**

```
checks.....................: 100.00% ✓ 1889822 ✗ 0      
data_received..............: 1.4 GB  1.8 MB/s
data_sent..................: 2.9 GB  3.8 MB/s
http_req_blocked...........: avg=4.42µs  min=1.05µs med=3.48µs  max=22.64ms  p(90)=6.91µs  p(95)=8.05µs 
http_req_connecting........: avg=15ns    min=0s     med=0s      max=2.18ms   p(90)=0s      p(95)=0s     
http_req_duration..........: avg=8.01ms  min=1.5ms  med=7.18ms  max=181.56ms p(90)=12.78ms p(95)=14.97ms
http_req_receiving.........: avg=52.24µs min=9.91µs med=43.63µs max=28.59ms  p(90)=82.87µs p(95)=98.81µs
http_req_sending...........: avg=30.53µs min=8.02µs med=25.41µs max=33.79ms  p(90)=49.79µs p(95)=58.5µs 
http_req_tls_handshaking...: avg=0s      min=0s     med=0s      max=0s       p(90)=0s      p(95)=0s     
http_req_waiting...........: avg=7.93ms  min=1.46ms med=7.1ms   max=181.33ms p(90)=12.7ms  p(95)=14.89ms
http_reqs..................: 1890822 2473.384895/s
iteration_duration.........: avg=8.38ms  min=1.64ms med=7.54ms  max=1m44s    p(90)=13.15ms p(95)=15.33ms
iterations.................: 1889822 2472.076795/s
script_errors..............: 0.00%   ✓ 0       ✗ 1889822
vus........................: 1       min=0     max=25   
vus_max....................: 25      min=25    max=25   
```

## Scenario "PBKDF2 with 1 iteration"

### Preparation

```sh
ansible-playbook -i setup/inventory setup/provision.yaml -e @setup/scenarios/baseline.yaml
rm -f k6/data/*.json
kci generate --realms 5 --clients 100 --users 1000 --template realm-templates/pbkdf-1it.template --target k6/data/
kci import k6/data/*.json
```

### Test: Login

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
REALM_COUNT=5 VU_COUNT=15 k6 run -o datadog k6/login.js
```

**Results**

```
checks.....................: 100.00% ✓ 313779 ✗ 0     
data_received..............: 1.1 GB  1.6 MB/s
data_sent..................: 292 MB  442 kB/s
http_req_blocked...........: avg=423.28µs min=1.27µs  med=7.62µs  max=51.92ms p(90)=1.35ms   p(95)=1.66ms  
http_req_connecting........: avg=386.42µs min=0s      med=0s      max=28.86ms p(90)=1.26ms   p(95)=1.56ms  
http_req_duration..........: avg=29.01ms  min=2.36ms  med=23.73ms max=1s      p(90)=56.47ms  p(95)=69.42ms 
http_req_receiving.........: avg=206.92µs min=16.22µs med=181.4µs max=18.9ms  p(90)=322.64µs p(95)=366.66µs
http_req_sending...........: avg=62.41µs  min=9.51µs  med=49.18µs max=14.41ms p(90)=135.11µs p(95)=158.33µs
http_req_tls_handshaking...: avg=0s       min=0s      med=0s      max=0s      p(90)=0s       p(95)=0s      
http_req_waiting...........: avg=28.74ms  min=2.07ms  med=23.47ms max=1s      p(90)=56.22ms  p(95)=69.16ms 
http_reqs..................: 313779  475.366144/s
iteration_duration.........: avg=90.39ms  min=14.08ms med=85.17ms max=2.11s   p(90)=135.09ms p(95)=151.49ms
iterations.................: 104593  158.455381/s
script_errors..............: 0.00%   ✓ 0      ✗ 104593
vus........................: 1       min=1    max=15  
vus_max....................: 15      min=15   max=15  
```

## Scenario "Only one Node"

### Preparation

```sh
ansible-playbook -i setup/inventory setup/provision.yaml -e @setup/scenarios/one-node.yaml
rm -f k6/data/*.json
kci generate --realms 5 --clients 100 --users 1000 --template realm-templates/baseline.template --target k6/data/
kci import k6/data/*.json
```

### Test: Login

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-1'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-1'
REALM_COUNT=5 VU_COUNT=10 k6 run -o datadog k6/login.js
```

**Results**

```
checks.....................: 100.00% ✓ 43131 ✗ 0    
data_received..............: 147 MB  223 kB/s
data_sent..................: 40 MB   61 kB/s
http_req_blocked...........: avg=325.96µs min=1.36µs   med=8.67µs   max=28.03ms  p(90)=1.04ms   p(95)=1.1ms   
http_req_connecting........: avg=279.53µs min=0s       med=0s       max=14.43ms  p(90)=894.39µs p(95)=946.54µs
http_req_duration..........: avg=145.23ms min=1.96ms   med=37.12ms  max=771.9ms  p(90)=440.57ms p(95)=486.04ms
http_req_receiving.........: avg=213.39µs min=27.21µs  med=194.01µs max=18.1ms   p(90)=333.24µs p(95)=366.26µs
http_req_sending...........: avg=82.89µs  min=11.31µs  med=70.68µs  max=1.54ms   p(90)=171.73µs p(95)=190.22µs
http_req_tls_handshaking...: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s      
http_req_waiting...........: avg=144.93ms min=1.66ms   med=36.88ms  max=771.55ms p(90)=440.31ms p(95)=485.7ms 
http_reqs..................: 43131   65.331743/s
iteration_duration.........: avg=439.19ms min=102.57ms med=446.01ms max=843.74ms p(90)=571.72ms p(95)=604.19ms
iterations.................: 14377   21.777248/s
script_errors..............: 0.00%   ✓ 0     ✗ 14377
vus........................: 1       min=1   max=10 
vus_max....................: 10      min=10  max=10 
```

### Test: Userinfo

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-1'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-1'
SESSION_COUNT=1000 REALM_COUNT=5 VU_COUNT=20 k6 run -o datadog k6/userinfo.js
```

**Results**

```
checks.....................: 100.00% ✓ 2318745 ✗ 0      
data_received..............: 783 MB  1.0 MB/s
data_sent..................: 3.0 GB  4.0 MB/s
http_req_blocked...........: avg=4.8µs   min=979ns  med=4.1µs   max=46.88ms  p(90)=7.26µs  p(95)=8.31µs  
http_req_connecting........: avg=7ns     min=0s     med=0s      max=1.54ms   p(90)=0s      p(95)=0s      
http_req_duration..........: avg=5.17ms  min=1.31ms med=4.73ms  max=147.89ms p(90)=8.2ms   p(95)=9.46ms  
http_req_receiving.........: avg=57.88µs min=9.54µs med=52.74µs max=18.77ms  p(90)=89.84µs p(95)=104.46µs
http_req_sending...........: avg=29.19µs min=5.96µs med=26.18µs max=19.54ms  p(90)=46.28µs p(95)=53.58µs 
http_req_tls_handshaking...: avg=0s      min=0s     med=0s      max=0s       p(90)=0s      p(95)=0s      
http_req_waiting...........: avg=5.09ms  min=1.26ms med=4.65ms  max=147.61ms p(90)=8.11ms  p(95)=9.37ms  
http_reqs..................: 2319745 3056.508899/s
iteration_duration.........: avg=5.47ms  min=1.43ms med=5.02ms  max=1m38s    p(90)=8.48ms  p(95)=9.74ms  
iterations.................: 2318745 3055.191294/s
script_errors..............: 0.00%   ✓ 0       ✗ 2318745
vus........................: 1       min=0     max=20   
vus_max....................: 20      min=20    max=20   
```

### Test: Refresh Token

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-1'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-1'
SESSION_COUNT=1000 REALM_COUNT=5 VU_COUNT=10 k6 run -o datadog k6/refresh-token.js
```

**Results**

```
checks.....................: 100.00% ✓ 384847 ✗ 0     
data_received..............: 1.2 GB  1.6 MB/s
data_sent..................: 368 MB  487 kB/s
http_req_blocked...........: avg=8.08µs   min=1.27µs  med=7.57µs   max=3.11ms   p(90)=10.53µs  p(95)=11.74µs 
http_req_connecting........: avg=21ns     min=0s      med=0s       max=918.57µs p(90)=0s       p(95)=0s      
http_req_duration..........: avg=15.86ms  min=4.78ms  med=14.4ms   max=107.78ms p(90)=23.32ms  p(95)=26.12ms 
http_req_receiving.........: avg=160.95µs min=16.64µs med=148.24µs max=12.15ms  p(90)=229.66µs p(95)=264.68µs
http_req_sending...........: avg=56.39µs  min=8.68µs  med=56.48µs  max=10.12ms  p(90)=76.5µs   p(95)=83.81µs 
http_req_tls_handshaking...: avg=0s       min=0s      med=0s       max=0s       p(90)=0s       p(95)=0s      
http_req_waiting...........: avg=15.64ms  min=4.66ms  med=14.18ms  max=107.66ms p(90)=23.1ms   p(95)=25.9ms  
http_reqs..................: 385847  510.276485/s
iteration_duration.........: avg=16.63ms  min=4.99ms  med=15.13ms  max=1m35s    p(90)=23.92ms  p(95)=26.72ms 
iterations.................: 384847  508.954001/s
script_errors..............: 0.00%   ✓ 0      ✗ 384847
vus........................: 0       min=0    max=10  
vus_max....................: 10      min=10   max=10  
```

### Test: Tokeninfo

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-1'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-1'
SESSION_COUNT=1000 REALM_COUNT=5 VU_COUNT=25 k6 run -o datadog k6/tokeninfo.js
```

**Results**

```
checks.....................: 100.00% ✓ 3711823 ✗ 0      
data_received..............: 2.5 GB  3.3 MB/s
data_sent..................: 5.4 GB  7.0 MB/s
http_req_blocked...........: avg=2.95µs  min=1.04µs med=2.34µs  max=16.48ms  p(90)=4.01µs  p(95)=4.81µs 
http_req_connecting........: avg=9ns     min=0s     med=0s      max=13.15ms  p(90)=0s      p(95)=0s     
http_req_duration..........: avg=4.02ms  min=1.04ms med=3.19ms  max=937.65ms p(90)=7.07ms  p(95)=8.72ms 
http_req_receiving.........: avg=34.12µs min=8.66µs med=29.34µs max=41.3ms   p(90)=47.03µs p(95)=55.97µs
http_req_sending...........: avg=20µs    min=6.91µs med=16.8µs  max=22.78ms  p(90)=28.44µs p(95)=34.13µs
http_req_tls_handshaking...: avg=0s      min=0s     med=0s      max=0s       p(90)=0s      p(95)=0s     
http_req_waiting...........: avg=3.97ms  min=1.01ms mkci generate --realms 1 --clients 10 --users 1000 --template realm-templates/baseline.template --target k6/data/
ed=3.14ms  max=937.29ms p(90)=7.01ms  p(95)=8.66ms 
http_reqs..................: 3712823 4841.367263/s
iteration_duration.........: avg=4.26ms  min=1.16ms med=3.43ms  max=1m46s    p(90)=7.31ms  p(95)=8.96ms 
iterations.................: 3711823 4840.063304/s
script_errors..............: 0.00%   ✓ 0       ✗ 3711823
vus........................: 1       min=0     max=25   
vus_max....................: 25      min=25    max=25   
```

## Scenario "Size S"

### Preparation

```sh
ansible-playbook -i setup/inventory setup/provision.yaml -e @setup/scenarios/baseline.yaml
rm -f k6/data/*.json
kci generate --realms 1 --clients 10 --users 100 --template realm-templates/baseline.template --target k6/data/
kci import k6/data/*.json
```

### Test: Login

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
REALM_COUNT=1 VU_COUNT=10 k6 run -o datadog k6/login.js
```

**Results**

```
checks.....................: 100.00% ✓ 34728 ✗ 0    
data_received..............: 118 MB  179 kB/s
data_sent..................: 32 MB   49 kB/s
http_req_blocked...........: avg=316.36µs min=1.39µs   med=8.53µs   max=15.98ms p(90)=1.03ms   p(95)=1.09ms  
http_req_connecting........: avg=275.76µs min=0s       med=0s       max=15.91ms p(90)=883.35µs p(95)=949.78µs
http_req_duration..........: avg=180.81ms min=2.53ms   med=40.92ms  max=1.27s   p(90)=626.96ms p(95)=754.73ms
http_req_receiving.........: avg=205.46µs min=26.43µs  med=186.58µs max=11.17ms p(90)=325.28µs p(95)=360.12µs
http_req_sending...........: avg=72.44µs  min=10.62µs  med=63.58µs  max=1.59ms  p(90)=157.87µs p(95)=184.25µs
http_req_tls_handshaking...: avg=0s       min=0s       med=0s       max=0s      p(90)=0s       p(95)=0s      
http_req_waiting...........: avg=180.53ms min=2.27ms   med=40.69ms  max=1.27s   p(90)=626.64ms p(95)=754.48ms
http_reqs..................: 34728   52.613047/s
iteration_duration.........: avg=545.59ms min=104.56ms med=532.37ms max=1.31s   p(90)=870.71ms p(95)=943.89ms
iterations.................: 11576   17.537682/s
script_errors..............: 0.00%   ✓ 0     ✗ 11576
vus........................: 1       min=1   max=10 
vus_max....................: 10      min=10  max=10 
```

### Test: Userinfo

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
SESSION_COUNT=1000 REALM_COUNT=1 VU_COUNT=20 k6 run -o datadog k6/userinfo.js
```

**Results**

```
checks.....................: 100.00% ✓ 2671233 ✗ 0      
data_received..............: 901 MB  1.2 MB/s
data_sent..................: 3.5 GB  4.5 MB/s
http_req_blocked...........: avg=4.28µs  min=969ns  med=3.58µs  max=25.21ms  p(90)=6.36µs  p(95)=7.35µs 
http_req_connecting........: avg=10ns    min=0s     med=0s      max=11.04ms  p(90)=0s      p(95)=0s     
http_req_duration..........: avg=4.49ms  min=1.28ms med=3.51ms  max=184.81ms p(90)=7.53ms  p(95)=9.48ms 
http_req_receiving.........: avg=51.55µs min=8.84µs med=45.36µs max=55.8ms   p(90)=78.2µs  p(95)=91.76µs
http_req_sending...........: avg=26.03µs min=5.69µs med=22.75µs max=27.85ms  p(90)=40.25µs p(95)=47.77µs
http_req_tls_handshaking...: avg=0s      min=0s     med=0s      max=0s       p(90)=0s      p(95)=0s     
http_req_waiting...........: avg=4.41ms  min=1.24ms med=3.44ms  max=184.48ms p(90)=7.45ms  p(95)=9.4ms  
http_reqs..................: 2672233 3496.662758/s
iteration_duration.........: avg=4.75ms  min=1.4ms  med=3.77ms  max=1m44s    p(90)=7.78ms  p(95)=9.73ms 
iterations.................: 2671233 3495.354241/s
script_errors..............: 0.00%   ✓ 0       ✗ 2671233
vus........................: 1       min=0     max=20   
vus_max....................: 20      min=20    max=20   
```

### Test: Refresh Token

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
SESSION_COUNT=1000 REALM_COUNT=1 VU_COUNT=10 k6 run -o datadog k6/refresh-token.js
```

**Results**

```
checks.....................: 100.00% ✓ 305927 ✗ 0     
data_received..............: 990 MB  1.3 MB/s
data_sent..................: 293 MB  384 kB/s
http_req_blocked...........: avg=9.32µs   min=1.26µs  med=8.73µs   max=3.3ms    p(90)=11.46µs  p(95)=12.58µs 
http_req_connecting........: avg=29ns     min=0s      med=0s       max=950.64µs p(90)=0s       p(95)=0s      
http_req_duration..........: avg=20.01ms  min=6.06ms  med=18.66ms  max=179.63ms p(90)=28.73ms  p(95)=32.14ms 
http_req_receiving.........: avg=171.99µs min=18.82µs med=147.76µs max=15.06ms  p(90)=247.91µs p(95)=297.87µs
http_req_sending...........: avg=62.38µs  min=9.01µs  med=58.81µs  max=14.53ms  p(90)=78.7µs   p(95)=87.01µs 
http_req_tls_handshaking...: avg=0s       min=0s      med=0s       max=0s       p(90)=0s       p(95)=0s      
http_req_waiting...........: avg=19.77ms  min=5.96ms  med=18.42ms  max=179.3ms  p(90)=28.5ms   p(95)=31.91ms 
http_reqs..................: 306927  402.864926/s
iteration_duration.........: avg=20.95ms  min=6.3ms   med=19.52ms  max=1m41s    p(90)=29.52ms  p(95)=32.8ms  
iterations.................: 305927  401.55235/s
script_errors..............: 0.00%   ✓ 0      ✗ 305927
vus........................: 1       min=0    max=10  
vus_max....................: 10      min=10   max=10  
```

### Test: Tokeninfo

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
SESSION_COUNT=1000 REALM_COUNT=1 VU_COUNT=25 k6 run -o datadog k6/tokeninfo.js
```

**Results**

```
checks.....................: 100.00% ✓ 2027454 ✗ 0      
data_received..............: 1.4 GB  1.8 MB/s
data_sent..................: 2.9 GB  3.8 MB/s
http_req_blocked...........: avg=5.37µs  min=1.1µs  med=4.69µs  max=14.11ms  p(90)=7.83µs  p(95)=8.88µs  
http_req_connecting........: avg=10ns    min=0s     med=0s      max=1.24ms   p(90)=0s      p(95)=0s      
http_req_duration..........: avg=7.37ms  min=1.46ms med=6.6ms   max=183.04ms p(90)=11.82ms p(95)=13.76ms 
http_req_receiving.........: avg=61.91µs min=9.71µs med=55.27µs max=38.09ms  p(90)=91.58µs p(95)=109.82µs
http_req_sending...........: avg=36.66µs min=7.01µs med=33.49µs max=52.38ms  p(90)=55.32µs p(95)=64.19µs 
http_req_tls_handshaking...: avg=0s      min=0s     med=0s      max=0s       p(90)=0s      p(95)=0s      
http_req_waiting...........: avg=7.27ms  min=1.39ms med=6.51ms  max=182.77ms p(90)=11.72ms p(95)=13.65ms 
http_reqs..................: 2028454 2657.165051/s
iteration_duration.........: avg=7.81ms  min=1.65ms med=7.04ms  max=1m43s    p(90)=12.24ms p(95)=14.18ms 
iterations.................: 2027454 2655.855105/s
script_errors..............: 0.00%   ✓ 0       ✗ 2027454
vus........................: 1       min=0     max=25   
vus_max....................: 25      min=25    max=25   
```

## Scenario "Size L"

### Preparation

```sh
ansible-playbook -i setup/inventory setup/provision.yaml -e @setup/scenarios/baseline.yaml
rm -f k6/data/*.json
kci generate --realms 10 --clients 1000 --users 10000 --template realm-templates/baseline.template --target k6/data/
kci import k6/data/*.json
```

### Test: Login

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
REALM_COUNT=10 VU_COUNT=10 k6 run -o datadog k6/login.js
```

**Results**

```
checks.....................: 100.00% ✓ 31407 ✗ 0    
data_received..............: 107 MB  162 kB/s
data_sent..................: 29 MB   44 kB/s
http_req_blocked...........: avg=265.81µs min=1.32µs   med=4.15µs   max=13.89ms p(90)=820.15µs p(95)=983.29µs
http_req_connecting........: avg=239.14µs min=0s       med=0s       max=13.81ms p(90)=735.3µs  p(95)=884.35µs
http_req_duration..........: avg=200.4ms  min=2.65ms   med=60.53ms  max=1.32s   p(90)=650.81ms p(95)=780.46ms
http_req_receiving.........: avg=181.21µs min=28.44µs  med=168.36µs max=5.56ms  p(90)=263.93µs p(95)=306.55µs
http_req_sending...........: avg=45.12µs  min=10.91µs  med=33.61µs  max=3.86ms  p(90)=76.06µs  p(95)=98.97µs 
http_req_tls_handshaking...: avg=0s       min=0s       med=0s       max=0s      p(90)=0s       p(95)=0s      
http_req_waiting...........: avg=200.18ms min=2.36ms   med=60.29ms  max=1.32s   p(90)=650.65ms p(95)=780.14ms
http_reqs..................: 31407   47.541077/s
iteration_duration.........: avg=603.38ms min=108.18ms med=595.91ms max=1.4s    p(90)=928.85ms p(95)=1s      
iterations.................: 10469   15.847026/s
script_errors..............: 0.00%   ✓ 0     ✗ 10469
vus........................: 1       min=1   max=10 
vus_max....................: 10      min=10  max=10 
```

## Scenario "Users in LDAP"

### Preparation

```sh
ansible-playbook -i setup/inventory setup/provision.yaml -e @setup/scenarios/ldap.yaml
rm -f k6/data/*.json
kci generate --realms 5 --clients 100 --users 0 --template realm-templates/ldap.template --target k6/data/
kci import k6/data/*.json
kci generate --realms 5 --clients 100 --users 1000 --template realm-templates/ldap.template --target k6/data/
```

### Test: Login

```sh
ansible -i setup/inventory sut -m shell -a 'podman stop keycloak-server-{1,2}'
ansible -i setup/inventory sut -m shell -a 'podman start keycloak-server-{1,2}'
REALM_COUNT=5 VU_COUNT=10 k6 run -o datadog k6/login.js
```

**Results**

```
checks.....................: 100.00% ✓ 275625 ✗ 0    
data_received..............: 942 MB  1.4 MB/s
data_sent..................: 256 MB  388 kB/s
http_req_blocked...........: avg=391.95µs min=1.29µs  med=7.63µs   max=25.79ms  p(90)=1.27ms   p(95)=1.55ms  
http_req_connecting........: avg=356.22µs min=0s      med=0s       max=25.74ms  p(90)=1.18ms   p(95)=1.45ms  
http_req_duration..........: avg=21.8ms   min=2.34ms  med=17.02ms  max=215.68ms p(90)=43.73ms  p(95)=55.99ms 
http_req_receiving.........: avg=208.82µs min=17.3µs  med=183.33µs max=22.76ms  p(90)=323.4µs  p(95)=365.95µs
http_req_sending...........: avg=61.61µs  min=9.3µs   med=48.37µs  max=13.06ms  p(90)=135.88µs p(95)=159.89µs
http_req_tls_handshaking...: avg=0s       min=0s      med=0s       max=0s       p(90)=0s       p(95)=0s      
http_req_waiting...........: avg=21.53ms  min=2.06ms  med=16.77ms  max=215.47ms p(90)=43.47ms  p(95)=55.71ms 
http_reqs..................: 275625  417.536918/s
iteration_duration.........: avg=68.63ms  min=15.86ms med=64.02ms  max=344.32ms p(90)=104.38ms p(95)=118.38ms
iterations.................: 91875   139.178973/s
script_errors..............: 0.00%   ✓ 0      ✗ 91875
vus........................: 1       min=1    max=10 
vus_max....................: 10      min=10   max=10 
```
