# Welcome to open source secret scanner

## How to run the entire application

**This application consists of 5 different modules.**

![Screenshot 2023-12-01 at 12.25.22 AM.png](images%2FScreenshot%202023-12-01%20at%2012.25.22%20AM.png)

1. [Frontend](https://github.com/hsivakum/opss) - 
    * A simple UI portal where the scan will be queued.
    * Both private and public repositories can be configured in this UI
    * For public repository we don't have to configure any personal access token.
    * For private repository user has to create a personal token in the respective SCM to schedule the scan
2. [Scan migrate](https://github.com/hsivakum/scan-service) - [![scan-service](https://github.com/hsivakum/scan-service/actions/workflows/ci.yml/badge.svg)](https://github.com/hsivakum/scan-service/actions/workflows/ci.yml)
    * a db migration docker image to create necessary database tables in the given target db connection
3. [Scan service](https://github.com/hsivakum/scan-service) - [![scan-service](https://github.com/hsivakum/scan-service/actions/workflows/ci.yml/badge.svg)](https://github.com/hsivakum/scan-service/actions/workflows/ci.yml)
    * A simple microservice written in go programming language
    * this service will expose a REST endpoint /api/v1/scan, and when frontend hits this endpoint scan will be inserted
      into the scan_requests table
    * after inserting the record, scan-service will push the data into the a kafka queue topic named scan-requested.
4. [Scan queue processor](https://github.com/hsivakum/scan-queue-processor) - [![scan-queue-processor](https://github.com/hsivakum/scan-queue-processor/actions/workflows/ci.yml/badge.svg)](https://github.com/hsivakum/scan-queue-processor/actions/workflows/ci.yml)
    * A simple microservice written in golang
    * this service will start consuming the data from the kafka queue topic scan-requested
    * once it consumes the data, it will make call to the github REST API to find the requested repo metadata such as
      size, branches etc.,
    * based on the size, it will create a persistence volume and persistent volume claim and pod in the given kubernetes
      cluster
    * after making resource allocation in the kubernetes truffle-scanner will scan the repo detailed provided in the POD
      env variables.
    * Change the status in the database from queued to processed
5. [Truffle scanner](https://github.com/hsivakum/truffle-scanner) - [![truffle-scanner](https://github.com/hsivakum/truffle-scanner/actions/workflows/ci.yml/badge.svg)](https://github.com/hsivakum/truffle-scanner/actions/workflows/ci.yml)
    * A simple go microservices written in golang
    * consume required details from environment variables.
    * if the repo is private then using git cli tool, it will clone the repo in certain directory and run the trufflehog
      executable to scan the cloned reop.
    * if public it will use the git schema runner from trufflehog
    * once the scan is complete, prase the output and writes and output to the database by batch.
6. [github-ss-etl](https://github.com/hsivakum/github-ss-etl) - 
    * A simple go tool written in golang
    * uses github go sdk to pull secret scanning data from given repository

## Running all services individually takes huge effort. Instead we will use docker to run all services under docker.

Pre Requisites:

1. Install docker in your machine
2. Create postgres database in azure and expose network publicly
3. Create azure key vault to store secrets required to use secrets securely
    * Create azure subscription.
    * Create key vault
    * Create a app registration with client id and client secret
    * Give necessary permissions like Owner, Contributor, Key Vault Contributor, Key Vault Crypto Officer, Key Vault
      Secrets User under Access Control in azure portal for the registered App.
4. Create below secrets
    * DB-PASSWORD - db password will be fetched in all microservices.
    * GITHUB-API-TOKEN - create a github api token for checking metadata of the public github repos.
    * SS-TOKEN - token for the repos to fetch secret scanning.
5. Since the runner requires kubernetes to run the scanner pod, use minikube to install free kubernetes
    * Create neccassary resources in kubernetes and update the values in global-secrets.yaml according to your
      configuration.
      ```kubectl create namespace truffle-scanner```
      ```kubectl -n truffle-scanner apply -f global-secret.yaml```
6. Copy the kubernetes config into config folder.
    * you can find the kubernetes config in root folder of your system, if windows then C:/Users/<username>/.kube/config
      if linux then ~/.kube/config
7. Provide all necessary values in the .env file.
8. run ```docker compose up -d zookeeper kafka scanner-db```
9. run ```docker compose up -d scan-migrate scan-service scan-queue-processor opss```
10. open http://localhost:3000 and try to schedule to scan.
11. enter the values and click on the submit
12. run the below command to see the pods running in the kubernetes which will automatically clone the repo, run the
    scan and upload the results to db.
    * ```kubectl -n truffle-scanner get pods```

**Note: If some of the above process looks complicated to run, ask the repo owner for their credentials and cloud specifications for demo purpose**

