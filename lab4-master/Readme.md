# Lab 4 - policy sync
> In this lab, we are going to reuse components from the previous lab and build on top of it. You should be familiar with most of the components.
> We are going to put ASM policy to separate repository, and have it synced with BIG-IP

## Review what is prepared
  * Base of repo is prepared, lets review it:
    * `/host_vars/bigip1.yml`
    * `1-requirements.yml`
    * `2-config-ltm-template.yml`
    * `3-test-vars.yml`
    * `ansible.cfg`
    * `as3-templ.json`
    * `inventory.ini`
## Deploy new app
  * We are just missing vars file to create new app, lets create 'vars.yml':
  ```
partition: lab4
apps:
      - name: dsvw
        vsip: 10.1.10.106
        vsport: 80
        memberport: 8000
        monurl: '/'
        monreceive: '200'
        poolname: dsvw_pool
        members:
          - 10.1.10.17
          - 10.1.10.18
  ```
    * test the application

## Add security
### Create new repository
  * Access GitLab home page
  * Click "New Project" at right top part of the page
  ![newproject](images/newproject1.png)
  * Select `Create blank project`
    * Name it `Lab4-policy-store`
    * Visibility level: `Public`
  ![newproject2](images/newproject2.png)

### Add security policy
  * Open `Web IDE`
  * Create new file `dsvw-policy.json` with following content:
```
  {
    "policy": {
        "name": "f5app_demo",
        "description": "F5 App Demo Policy",
        "template": { "name": "POLICY_TEMPLATE_FUNDAMENTAL" },
        "applicationLanguage": "utf-8",
        "enforcementMode": "blocking",
        "blocking-settings": {
                "violations": [
                    {
                    "name": "VIOL_METHOD",
                    "block": true,
                    "description": "Block unlisted method options"
                    },
                    {
                    "name": "VIOL_FILETYPE",
                    "block": true,
                    "description": "Block unlisted file options"
                    }
                ]
        },
        "signature-sets": [
            {
                "name": "Generic Detection Signatures (High/Medium Accuracy)",
                "block": true,
                "alarm": true
            }
        ],
        "data-guard": {
            "enabled": "true",
            "maskData": "true",
            "creditCardNumbers": true
        },
        "whitelist-ips": [
            {
                "ipAddress": "24.200.200.2",
                "ipMask": "255.255.255.255",
                "description": "allow static",
                "blockRequests": "never",
                "trustedByPolicyBuilder": true
            },
            {
                "ipAddress": "24.100.100.1",
                "ipMask": "255.255.255.255",
                "description": "allow static",
                "blockRequests": "never",
                "trustedByPolicyBuilder": true
            }
        ],
        "methods": [
            {
                "name": "DELETE"
            },
            {
                "name": "OPTIONS"
            }
        ],
        "filetypes": [
            {
                "name": "svg",
                "allowed": false
            }
        ],
        "disallowed-geolocations": [
            {
                "countryName": "American Samoa"
            },
            {
                "countryName": "Algeria"
            },
            {
                "countryName": "Albania"
            }
        ]
    }
}

```
  * And `Commit`

### Attach security policy to existing app

  * Go to Lab4 repo and modify `vars.yml` so it refers to newly created policy:
```
partition: lab4
apps:
      - name: dsvw
        vsip: 10.1.10.106
        vsport: 80
        memberport: 8000
        monurl: '/'
        monreceive: '200'
        poolname: dsvw_pool
        members:
          - 10.1.10.17
          - 10.1.10.18
        wafpolicyname: 'dsvw-policy'
        wafpolicyurl: 'http://10.1.10.250/gitlab-instance-371ef8c6/lab4-policy-store/-/raw/main/dsvw-policy.json'
        logprofile: '/Common/Log all requests'
```
  * `Commit` and check the results in BIG-IP

## BIG-IP to GitLab sync

  > We need to keep our policy in gitlab up to date, so when WAF admin updates the policy it will be automatically replicated to GitLab. Logical flow:

    * WAF Admin updates the policy
    * Policy creates webhook to GitLab repository
    * Webhook triggers pipeline
    * Pipeline makes GitLab Runner to download policy from BIG-IP and push it to the repo.

  > To enable this flow, we need to go from the bottom.

### GitLab settings

  * In Lab4-policy-store repository create `sync` file with following content:
```
#!/bin/bash

download_waf () {
  #find policyReferenc id
  lId=$(curl -sku $F5_USER:$F5_PASS https://${F5_IP}/mgmt/tm/asm/policies | jq -r ".[\"items\"][] | select(.[\"name\"] == \"${POLICY_NAME}\")[\"id\"]")
  #Prepare file for json format to download set check equal to task id 
  check=$(curl -sku $F5_USER:$F5_PASS -X POST -H "Content-Type: application/json" -d "{\"filename\": \"pol.json\",\"minimal\": true,\"format\": \"json\",\"policyReference\": {\"link\": \"https://localhost/mgmt/tm/asm/policies/${lId}\"}}" https://${F5_IP}/mgmt/tm/asm/tasks/export-policy | jq -r '.id')
  timer=0
  while [ COMPLETED != $(curl -sku $F5_USER:$F5_PASS https://${F5_IP}/mgmt/tm/asm/tasks/export-policy/${check} | jq -r '.status') ]
    do 
      timer=$((timer+1))
      if [ "$timer" = 54 ]; then
        echo "Time exceeded"
        exit -1
      fi
      echo "Seconds elapsed "$((timer * 10))
      sleep 10
    done
  #Download file 
  curl -sku $F5_USER:$F5_PASS https://${F5_IP}/mgmt/tm/asm/file-transfer/downloads/pol.json | jq > "${POLICY_NAME}.json"
}

download_waf
```
  * And `Commit`
  * Now we have to enable access to repository via SSH key.
  * Go to `Lab4-policy-store` repository, `Settings` -> `Repository` -> `Deploy Keys` and Add `SSH_PUSH_PUB` key with `wrtie` permissions and following content (or generate your own):
```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCvPFwfq+OZ3Sa4p/h3DL9YNkJkbYV9El8k9hbBm46QFddSBTI5mFtdAdJMfXr65qn4bYMqJWpXSpISulZQjAQf3G51ZlfWFa2jeUO37zFF3/RU0vFVIp8bxYVQot4r2gRau5GUDzmCi6vVoDm687p8Lx3s3YwqyGCCcxlt3KzFSccx4dQ1dRCLDJ6VzZEmW95zcNKeWR0AjEr5Di1ecPhPi3GRDTTv5WnpGMZ+ReX1BxRSzI6qnW7CciAwVvLdsl7KZcSsh2NEAjVNB3fhbBApj1BGrbvZ6o1RZotgCFPjLoZ+XRz7cLWBvQJuk4dJWLmS3ohCbCnRcY4B/RIhozBj3VnfDpp+LYJkrof2kbjiSA5la/BARZEnphNNS2gF0g4DxkGAgozMqJIPcBuvtLXtoXMgb/KZcxX0wbCxEOJUJkL0GRG2KzXzKc2eOu7wzHKQQPxYvJ9ymjiAKKzR1PiSEufYzVZtNcvjMGNADFD4voi0RdV+4J0lxeOX0k7ZibAfZS0Ca/PmEDOeJnuEbGnXf5dBDFJJRB7keHjHk66YFAp9irtJtWj7Sm7q9UPur/pKPHcgkVSKBtXegglPjDsEr5XGfUEww0gXFu9dLJ7bMTQkdbmDeQlo6+5bN1pcXTN4enM1jEgFQ7J6VU98yA7LbPe6Ur684zkBHWODlJtmDw== ubuntu@ubuntu
```
  * It allows everyone with correct private key to update the repo
![addkey](images/addkey.png)

  * Go to `Lab4-policy-store` repository, `Settings` -> `CI/CD` -> `Variables` and create following variables:
![addvar](images/addvar.png)
  * Dont protect the variables, its just a lab :)
![vardetail](images/vardetail.png)
  * Variables content:
    * `F5_USER` `admin`
    * `F5_PASS` `heslohesel123`
    * `SSH_PUSH_KEY`
```
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```
  * Now script knows how to connect to BIG-IP and have access to private key which can update the repository.
  * We need to add `Pipeline trigger` so BIG-IP can start the pipeline. Just below `Variables` are `Pipeline triggers`, lets create one and note the value.
![trigger](images/trigger.png)
### Create pipeline
  * Pipeline as presented below `!!! SHOULD NEVER BE USED IN ANY REASONABLE ENVIRONMENT !!!` proper use is described in [original article](https://community.f5.com/t5/technical-articles/manage-f5-big-ip-advanced-waf-configuration-drift-with-webhooks/ta-p/300988)
  * In `Lab4-policy-store` define pipeline - `CI/CD` -> `Editor`:
  ```
variables:
  GIT_STRATEGY: clone

stages:
  - pull

Config:
  stage: pull 
  before_script:
    - echo $USER
    - ssh-keyscan 10.1.10.250 > ~/.ssh/known_hosts
    - echo "${SSH_PUSH_KEY}" > ~/.ssh/id_rsa
    - git config user.email "user@udf.com"
    - git config user.name "udfuser"
    # - git remote add ssh_origin "git@$CI_SERVER_HOST:$CI_PROJECT_PATH.git"
    - git clone "http://localhost/$CI_PROJECT_PATH.git"
    - cd lab4-policy-store
    - git remote set-url origin http://root:heslohesel123@10.1.10.250/gitlab-instance-371ef8c6/lab4-policy-store.git
    - git config --global user.email "runner@udf.com"
    - git config --global user.name "runner"
    - export TZ=Europe/Prague
  script:
    - source ./sync
    - git status
    - git diff-index --quiet HEAD || git commit -am "sync waf_pol `date`"
    - git push origin HEAD:main
  retry: 2
  rules:
  - if: $CI_PIPELINE_SOURCE == "trigger"
    when: always
  timeout: 10m
  ```

### BIG-IP Settings

  * Login to bigip1 as admin/heslohesel123
  * Go to `Security` -> `Application Security` -> `Policies List` and switch to `Lab 4` partition:

  ![policy](images/asm.png)

  * open the policy, in `General Settings` scroll down to `Outgoing Webhooks` and `+ Add Webhook`:

![policy-wh](images/policy-wh.png)

  * Webhook Name: `waf-hook`
  * Trigger Event: `Apply Policy`
  * Webhook URL: `http://10.1.10.250/api/v4/projects/10/trigger/pipeline`
  * Content Type: `application/json`
  * Body - replace [your token] with token created in repository settings: {"token": "[your token]","ref": "main","variables": {"POLICY_NAME": "{{policy.name}}", "HOST_NAME": "{{device.hostname}}", "F5_IP": "10.1.1.4"}}

  ![webhook](images/webhook.png)
  
  * Apply policy
  * Modify policy, Apply it.
  * Check policy repository, pipeline, change policy, and again...

  ![done1](images/done1.png)
  ![done2](images/done2.png)