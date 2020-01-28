# CI-CD Workflow 
Feature
======
 - deploy base on git branch
 - strategy: multi-env on test env, staging , canary, rollout
 - support crd
   - All K8S CRD
 - configmap & secret in  will add suffix
    - configmapName: myconfig 
    - myconfig-dev   <<dev
    - myconfig-staging  <<staging
    - myconfig-canary  <<canary
    - myconfig-prod-v001 << prod add suffix to support rollback
 - support [advance feature](kubernetes-advance):  
   - 1 deployment -> multiple app. 
   - golden image
   - init job

Flow
------
### Development branch  
*e.g. /develop , /feature/feature001 , /hotfix/hotfix144 , /master*

build >> test >> quality >> build-docker & create-habor-project [source](2-docker-ci-template.yml)

<br/>

### Deployment branch: env/{envName} sit/{{env}}   sit-{{env}}   env-{{env}}

build >> test >> quality >> docker-image.build-push >> deploy-test 
[source](3-deploy-test-ci-template.yml)

<br/>

### Deployment branch: release/{version}/{ccNumber}
![Flow](doc/flow-prod-1.jpg)
![Flow](doc/flow-prod-2.jpg)

#### Release Flow
##### prepare phase
1. developer >> commit final code to development branch
2. owner >> branch code to release/{version}/{ccNumber}
##### approve phase phase
3. CI >> run job: build>> push to habor >> send email to CC. [source](4-start-workflow-template.yml)
4. CC >> login to system and approve flow [at bismarck](http://bismarck-prod.devops-wf.atlantic.true.th/)
##### pre-prod phase
5. owner >> now able to click deploy to staging [source](5-deploy-staging-ci-template.yml)
6. owner >> wait until staging period (as agree with CC) when click manual task to deploy staging.

7. user >> access {ingressName}-staging.{namespace}.atlantic.true.th to test
  * Application should detect environment by env=staging. ( your kustomize file shall set env=staging [value](kubernetes/overlay/staging/patch.yaml) )
```
        +-----------------------------------+
        |              Ingress              |
        | {ingressName}-staging.{namespace} |
        |                                   |
        +-----------------------------------+
                         |
                         v
                  +--------------+
                  |   service    |
                  | svc-staging  |
                  |              |
                  +--------------+
                         |
                         v
            +------------------------------+
            | Pod (deployment/statefulset) |
            |         xxx-staging          |
            |                              |
            +------------------------------+
```
8. user >> confirm test pass.
9. owner >> deploy canary job. [source](6-canary-ci-template.yml)
```
            +----------------------+         +----------------------+
            |                      |         |                      |
            | {ingressName}-canary |         | {ingressName}-prod   |
            |                      |         |                      |
            +----------+-----------+         +----------+-----------+
                       |                                |
                       v                                v
                +--------------+                 +--------------+
                |   service    |                 |   service    |
                | svc-canary   |  +--------------+ svc-prod     |
                |              |  |              |              |
                +------+-------+  |              +------+-------+
labels app: xxx-prod   |          |                     | labels app: xxx-prod
       mode: canary    v          v                     v
      +----------------+-------------+   +--------------+---------------+
      | Pod (deployment/statefulset) |   | Pod (deployment/statefulset) |
      |         xxx-canary           |   |         xxx-prod             |
      | labels app:xxx-pmode: canary |   | labels app:xxx-prod          |
      |        mode: canary          |   +------------------------------+
      +------------------------------+
```
10. real traffic >> will route to current production pod & new canary pod.
11. {ingressName}-canary traffic >> will route to only canary pod.
12. user/owner >> test to {ingressName}-canary to verify everything is ok.
13. owner >> inspect canary logs, verify everything ok.  
  ( your [kustomize file file](kubernetes/overlay/canary/patch.yaml) )  shall set
 * env=prod
 * Log level to trace
 
##### rollout prod phase
14. owner >> run rollout job [source](7-rollout-prod-ci-template.yml)
15. owner >> run rollback job, if there are some problem. [source](8-rollback-prod-ci-template.yml)
16. owner >> click close cc , after pass observation period.
##### cleanup prod phase
17. owner >> optinally: run job todelete sanity,canary deployment.
#### Release strategy
developer can skip staging,canary for some specific strategy.


 

## Background knowledge.
To able to develop flow, developer need to understand
 - k8s deployment CRD : ingress , service , [ deployment or statefulset]
 - k8s kustomize https://github.com/kubernetes-sigs/kustomize


## Gitlab-ci
### Stages
[source](1-base-ci-template.yml)

  | States   | Description |
  | --------|-------------|
  | build             |  Compile source code and build artifact |
  | test              |  Unit testing |
  | quality            |  Code quality checking #ex. sonarqube |
  | docker-image      |  Build to image (Docker) |
  | deploy-test        |  Deploy Cluster Dev |
  | start-workflow      | Start Workflow process for deploy production |
  | deploy-sanity     |  Deploy Cluster Prod |
  | automation-test     |  Future requirement |
  | rollout           |  Rollout  |
  | rollback           |  Rollback |


## Development.

### Prepare
1. Setup variable Create project structure. Recommend struture for reusable variable is:
```
└── {team-level-subproject}
    ├── HABOR_PASS **
    ├── HABOR_USER **
    ├── PROD_KUBECONFIG **
    ├── TEST_KUBECONFIG **  
    └── {project-level-subproject}       
        ├── {app_project}
        └── {app_project}
```
###### Example: 
 ![variable](doc/var.jpg) [..](https://gitlab.com/groups/bismarck-shared/ci-cd/demo-cicd/team-level/-/settings/ci_cd)

2. create project
you can check demo and fork project to run test [here](https://gitlab.com/bismarck-shared/ci-cd/demo-cicd/team-level/project-level/cicd-demo)
```
└─Dockerfile  # your dockerfile ( optional )
└─.gitlab-ci.yml  # see below
└─kubernetes
  ├── base
  │   ├── kustomization.yaml
  │   └── manifest.yaml  # your manifest file
  └── overlay
      ├── canary
      │   ├── kustomization.yaml
      │   └── patch.yaml
      ├── dev
      │   ├── kustomization.yaml
      │   └── patch.yaml
      ├── prod
      │   ├── kustomization.yaml
      │   └── patch.yaml
      └── staging
          ├── kustomization.yaml
          └── patch.yaml
```
###### Dockerfile
  - In case you have on-the-shelf image, you can omit this file. CI will skip build if not exist.

###### .gitlab-ci.yml
```yml
variables:
  namespace: "snail"  #*your namespace*

include:
  - project: 'bismarck-shared/ci-cd/standard-ci'
    ref: kustomize
    file: '1-base-ci-template.yml'
  - project: 'bismarck-shared/ci-cd/standard-ci'
    ref: kustomize
    file: '2-docker-ci-template.yml'
  - project: 'bismarck-shared/ci-cd/standard-ci'
    ref: kustomize
    file: '3-deploy-test-ci-template.yml'
  - project: 'bismarck-shared/ci-cd/standard-ci'
    ref: kustomize
    file: '4-start-workflow-template.yml'
  - project: 'bismarck-shared/ci-cd/standard-ci'
    ref: kustomize
    file: '5-deploy-staging-ci-template.yml'
  - project: 'bismarck-shared/ci-cd/standard-ci'
    ref: kustomize
    file: '6-canary-ci-template.yml'
  - project: 'bismarck-shared/ci-cd/standard-ci'
    ref: kustomize
    file: '7-rollout-prod-ci-template.yml'
  - project: 'bismarck-shared/ci-cd/standard-ci'
    ref: kustomize
    file: '8-rollback-prod-ci-template.yml'

build:
  tags:
    - docker-itsd
  stage: build
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    untracked: true
    policy: pull-push
  image: xxxxxx   #<=============== image for compile your source code
  script: 
    - echo build #<=============== your script

test:
  tags:
    - docker-itsd
  stage: test  
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    untracked: true
    policy: pull
  image: xxxxxx   #<=============== image for unit test your source code
  script: 
    - echo build #<=============== your script
```

###### k8s directory
must contain └─kubernetes/overlay/{$env,staging,canary,prod}/kustomization.yaml
  - $env is your git branch e.g.: if *env/uat1*  then *env=uat1*


3. Create secret for deployment.
```
kubectl create secret docker-registry regcred-harbor --docker-server=reghbpr01.dc1.true.th --docker-username=prod-eng-sa --docker-password=xxxx 
```
4. commit >> push to start ci.


## deployment configuration practice
your [manifest file](kubernetes/base/manifest.yaml) shall define below configuration.
### Resource  
- limit.memory, request.memory as your requirement.
- request.cpu : optional, default is 50m, define must more than 50m.
- limit.cpu: optional, default is 4 core.
### Probe 
 readinessProbe: kube wait until ready to send traffic.
 livenessProbe: kube detect healthy , terminate pod and launch new one.
### namespace:
- Due to we config kube-config at team level. manifest should define namespace (   manifest file or kustomize file).
### ingress:
- ingress host={ingressname}.{namespace}.arctic.true.th ( CI will patch host to atlantic on /release/*)

### kustomize pattern
#### Base file
- set ingress host based on ingress name
#### Overlay file
```
bases:
- ../../base
images:
- name: ${image}     🚩(1)
  newTag: ${imageTag}    🚩(1)
nameSuffix: -staging   🚩(2)
commonLabels:
  app: demo-ci-staging   🚩(3)
patchesStrategicMerge:
- patch.yaml   🚩(4)
```
1. 🚩CI will replace variable before run build
2. 🚩all object must siffix with -staging
3. 🚩set label for all object
4. 🚩change spec in object
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-ci 
spec:
  template:
    spec:
      containers:
      - name: primary
        env:
        - name: env
          value: staging  🚩 change env parameter.
        resources:
          limits:
            memory: 128Mi 🚩 set memory.
```

## Integrate your custom flow to CC Approve .
In case you have your custom flow and need to algin it to CC approval.
  1. When job request to CC call>>see [start-workflow](4-start-workflow-template.yml) 
  2. When request has been approved >> Bismarck flow will trigger job name:  [waiting-approve](4-start-workflow-template.yml) 
  3. each complete need to report to bismarck
```
    curl http://bismarck-prod.devops-wf.atlantic.true.th/completed/$CI_PIPELINE_ID/$stage/$GITLAB_USER_EMAIL
```
- possible stage is
    - staging
    - canary
    - rollout
    - rollback
    - close  #close cc

