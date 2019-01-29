## Gitlab Rancher deployment Example

In this example we will create an installation of Gitlab and Rancher from strach on Digitalocean.

Youtube recording https://www.youtube.com/watch?v=sHjnpWuJhwA&list=PLJ0nYE0NtQxaoo-KJ5ciDn2AbOGHlH3OI

### 1. Gitlab

#### 1.0. create Gitlab Digitalocean VM

````
docker-machine create --driver digitalocean --digitalocean-access-token $DOTOKEN --digitalocean-image ubuntu-16-04-x64 --digitalocean-size 4gb --digitalocean-tags gitlab-rancher-example gitlab-host
eval $(docker-machine env gitlab-host)
````

#### 1.1 start Gitlab UI
````
cd gitlab-ui
cat docker-compose.yml
docker-compose up -d
````


#### 1.2 start Gitlab CI runner

````
cd gitlab-runner
cat docker-compose.yml
docker-compose up -d
````


register gitlab-runner to the gitlab instance:
````
docker exec -it gitlabrunner_web_1 bash
gitlab-runner register
````
* coordinator URL: http://DOCKER_MACHINE_IP
* CI token: login as root in Gitlab UI >> admin area >> runners
* executor: docker
* default docker image: java:8-jdk

check registered runner on Gitlab UI: login as root in Gitlab UI >> admin area >> runners

#### 1.3 create a Gitlab user

#### 1.4 create a Gitlab repository
- Generate Gitlab Personal access token: Github >> Settings Personal Access tokens >> create
- Import from https://github.com/mariodavid/kubanische-kaninchenzuechterei

#### 1.5 configure Gitlab CI
Set up CI >> Gradle template

#### 1.6 run Gitlab CI pipeline

#### 1.7 build the container and push to dockerhub

.gitlab-ci.yml:

````
image: java:8

variables:
    GRADLE_OPTS: "-Dorg.gradle.daemon=false"
    CONTAINER_TEST_IMAGE: mariodavid/kubanische-kaninchenzuechterei:$CI_BUILD_REF_NAME
    CONTAINER_RELEASE_IMAGE: mariodavid/kubanische-kaninchenzuechterei:latest


before_script:
    - chmod +x gradlew


stages:
  - build
  - test
  - release-jar
  - release-container

build:
  stage: build
  script:
    - ./gradlew -g /cache/.gradle clean assemble
  allow_failure: false

test:
  stage: test
  script:
    - ./gradlew -g /cache/.gradle check

buildJar:
  stage: release-jar
  script:
    - ./gradlew -g /cache/.gradle buildUberJar
  artifacts:
    untracked: true

buildContainer:
  image: docker:git
  services:
    - docker:dind
  stage: release-container
  dependencies:
    - buildJar
  before_script:
    - docker login -u $DOCKER_REGISTRY_USERNAME -p $DOCKER_REGISTRY_PASSWORD
  script:
    - mkdir -p ./deployment/container-files
    - cp ./build/distributions/uberJar/app.jar ./deployment/container-files/app.jar
    - cd ./deployment
    - docker build -t $CONTAINER_TEST_IMAGE .
    - docker push $CONTAINER_TEST_IMAGE
  only:
    - master

````


#### 1.7 fix docker-in-docker problem in gitlab-runner
````
docker exec -it gitlabrunner_web_1 bash
nano /etc/gitlab-runner/config.toml
gitlab-runner restart
````
set the following attributes:
````
privileged = true
volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"]
````

### 2. Rancher

#### 2.0 create Rancher Digitalocean VM
````
docker-machine create --driver digitalocean --digitalocean-access-token $DOTOKEN --digitalocean-image ubuntu-16-04-x64 --digitalocean-size 2gb --digitalocean-tags gitlab-rancher-example rancher-ui-host
eval $(docker-machine env rancher-ui-host)
````

#### 2.1 start Rancher UI
````
cd rancher
cat docker-compose.yml
docker-compose up -d
````

#### 2.2 craete a Rancher host for the default environment to run our containers

Rancher UI > Default environment > Hosts > Add Host > Digitalocean. Add the Access token of Digitalocean ($DOTOKEN) and create a machine.

#### 2.3 create access control, environment & API token

Create a Rancher CLI access token through the UI of rancher (API > Keys) and put the following environment variables into your .bashrc file:
````
export RANCHER_URL=http://<<RANCHER_IP>>:8080
export RANCHER_ACCESS_KEY=<accessKey_of_account_api_key>
export RANCHER_SECRET_KEY=<secretKey_of_account_api_key>
````


#### 2.4 deploy application stack manually
````
git clone https://github.com/mariodavid/kubanische-kaninchenzuechterei
cd kubanische-kaninchenzuechterei/deployment
cat docker-compose.yml
cat rancher-compose.yml

rancher up
````


### 3. Deploy to Rancher from Gitlab

Add environment variables to the Gitlab project:
````
RANCHER_URL=http://RANCHER_HOST_IP:8080/
RANCHER_ACCESS_KEY=...
RANCHER_SECRET_KEY=...
````
Add the following step to your .gitlab-ci.yml:
````
stages:
  - ...
  - deploy

deploy:
  stage: deploy
  image: cdrx/rancher-gitlab-deploy
  script:
    - upgrade --stack deployment --service web --new-image $CONTAINER_TEST_IMAGE

````
