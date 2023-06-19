# Deoloy Laravel on AWS ECS with Dockerfile + AWS CodeBuild
## Requirements
1. A Windows, Linux or Mac PC with Docker installed
2. An AWS Account
3. Laravel
4. A Github Account

I will use AWS CodeBuild — “a fully managed continuous integration service that compiles source code, runs tests, and produces software packages that are ready to deploy” — to build our container image whichwill be stored on AWS Elastic Container Registry(ECR), an Amazon managed docker container registry — think Docker hub.

## Get Composer
Firstly, we need to download Composer, a PHP package manager. We will use Composer to install Laravel and other dependencies. 

## Architecture Overview
<img width="468" alt="image" src="![ArchitectureOverview](https://github.com/nholuongut/Deploy-Lavarel/assets/58627821/16751ed4-5be3-49b8-b4b4-8127b9bea051)">

## Build and run with docker-compose (local)

```bash
$ docker-compose up --build
```

### Build assets

```bash
$ docker-compose run --rm nodejs bash -c "npm install && npm run dev"
```

## Build and push docker to AWS ECR (Elastic Container Registry)

### Create ECR repository
Create 2 repositories for `nginx` and `php-fpm`.
Like below.

- `laravel-ecs-demo/nginx`
- `laravel-ecs-demo/php-fpm`

### docker login (with latest `awscli`)

```bash
$ pip install awscli --upgrade
$ $(aws ecr get-login --no-include-email --region ${AWS_DEFAULT_REGION})
```

### Build nodejs and assets(js/css)

```bash
$ docker build -t nodejs -f docker/nodejs/Dockerfile .
$ docker run --rm -v $(pwd):/build -w /build nodejs npm install
$ docker run --rm -v $(pwd):/build -w /build nodejs npm run dev
```

### Build php-fpm/nginx

```bash
$ docker build -t ${REPOSITORY_URI_PHP_FPM}:latest -f docker/php-fpm/Dockerfile .
$ docker build -t ${REPOSITORY_URI_NGINX}:latest -f docker/nginx/Dockerfile .
```

### Add tags

```bash
$ docker tag ${REPOSITORY_URI_PHP_FPM}:latest ${REPOSITORY_URI_PHP_FPM}:$IMAGE_TAG
$ docker tag ${REPOSITORY_URI_NGINX}:latest ${REPOSITORY_URI_NGINX}:$IMAGE_TAG
```

### Push docker image

```bash
$ docker push ${REPOSITORY_URI_PHP_FPM}:latest
$ docker push ${REPOSITORY_URI_PHP_FPM}:$IMAGE_TAG
$ docker push ${REPOSITORY_URI_NGINX}:latest
$ docker push ${REPOSITORY_URI_NGINX}:$IMAGE_TAG
```

## Create Task definition

<table>
  <tr>
    <th align="left">name</th><td>laravel-ecs-demo</td>
  </tr>
  <tr>
    <th align="left">Launch type</th><td>EC2</td>
  </tr>
</table>

### Add php-fpm container

<table>
  <tr>
    <th align="left">name</th><td>php-fpm</td>
  </tr>
  <tr>
    <th align="left">image (uri)</th><td>${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/php-fpm:latest</td>
  </tr>
  <tr>
    <th align="left">working directory</th><td>/app</td>
  </tr>
  <tr>
    <th align="left">memory</th><td>300</td>
  </tr>
</table>

### Add nginx container

<table>
  <tr>
    <th align="left">name</th><td>nginx</td>
  </tr>
  <tr>
    <th align="left">image (uri)</th><td>${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/nginx:latest</td>
  </tr>
  <tr>
    <th align="left">port mapping</th><td>tcp 80:80</td>
  </tr>
  <tr>
    <th align="left">memory</th><td>300</td>
  </tr>
  <tr>
    <th align="left">links</th><td>php-fpm</td>
  </tr>
</table>

## Create ECS Cluster

<table>
  <tr>
    <th align="left">name</th><td>laravel-ecs-demo-cluster</td>
  </tr>
  <tr>
    <th align="left">cluster compatibility</th><td>EC2 Linux + Networking</td>
  </tr>
  <tr>
    <th align="left">EC2 Instance type</th><td>t2.small</td>
  </tr>
  <tr>
    <th align="left">Provisioning model</th><td>Spot</td>
  </tr>
  <tr>
    <th align="left">Number of instances</th><td>2</td>
  </tr>
  <tr>
    <th align="left">Maximum bid price (per instance/hour)</th><td>10($)</td>
  </tr>
</table>

## Create ALB (Application load balancer)

<table>
  <tr>
    <th align="left">Availability Zone</th><td>(Choose from VPC of ECS Cluster)</td>
  </tr>
  <tr>
    <th align="left">Listeners</th><td>HTTP 80</td>
  </tr>
  <tr>
    <th align="left">Target instances</th><td>(Choose from Availability Zone)</td>
  </tr>
</table>

## Create Service in ECS Cluster

<table>
  <tr>
    <th align="left">Launch type</th><td>EC2</td>
  </tr>
  <tr>
    <th align="left">Task definition</th><td>laravel-ecs-demo</td>
  </tr>
  <tr>
    <th align="left">Cluster</th><td>laravel-ecs-demo-cluster</td>
  </tr>
  <tr>
    <th align="left">Service name</th><td>laravel-ecs-demo</td>
  </tr>
  <tr>
    <th align="left">Service type</th><td>REPLICA</td>
  </tr>
  <tr>
    <th align="left">Load balancer type</th><td>Application Load Balancer</td>
  </tr>
  <tr>
    <th align="left">Container to load balance</th><td>nginx:80:80</td>
  </tr>
  <tr>
    <th align="left">Listener port</th><td>80:HTTP</td>
  </tr>
  <tr>
    <th align="left">Target Group</th><td>(Choose of ALB)</td>
  </tr>
</table>
