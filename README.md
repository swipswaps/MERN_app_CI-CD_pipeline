# MERN application with CI-CD pipeline

## Continuous Delivery pipeline and monitoring for a MERN application

CI/CD pipelines allow for a fast response to business needs, while other DevOps practices support stable, secure and predictable services, striking the perfect balance  between stability and the speed of change.

This is a final project by Pedro Tavares for a DevOps Bootcamp course at [GeeksHubs Academy](https://geekshubsacademy.com/) (in spanish).

It creates a Continuous Delivery pipeline for a JavaScript/MERN application, with **development**, **staging** and **production** environments, tests and monitoring.

![CI/CD pipeline](https://github.com/ptavaressilva/MERN_app_CI-CD_pipeline/blob/master/img/pipeline.png?raw=true) 

### Requirements and coverage

- [x] The application must be containerized
- [x] The application must communicate with a database
- [x] Containers must be managed by an orchestrator
- [x] It must be possible to test the pipeline locally (e.g. not require a cloud provider account)
- [x] Pushing to master must trigger a pipeline that terminates with a deployment to production (Continuous Delivery)
- [x] The pipeline must include Development, Staging and Production environments
- [ ] The system must include application and infrastructure monitoring, and a dashboard

*The actual software application falls outside the scope of this project.*

### The Technological Stack

Currently the project uses the following technologies:

- MERN stack (MongoDB, Express, React and NodeJS)
- Docker (incl. Docker Compose and Swarm)
- Docker Registry
- Gogs
- Jenkins

This section will be updated as the project progresses.

### Application placeholder

The application placeholder was bootstrapped with [Create React App](https://github.com/facebook/create-react-app), the officially supported way to create single-page React applications.

The app was basd on [mern-docker-starter](https://github.com/joshdcuneo/mern-docker-starter.git). When started, it creates a user in the database. When a page is requested (to the client), the client calls the server API, which queries the database and returnss the username created at startup.

![The application](https://github.com/ptavaressilva/MERN_app_CI-CD_pipeline/blob/master/img/app_page.png?raw=true)

As **this application falls ouside the scope of the project**, its limitations were not addressed (e.g. it creates the same user every time it starts, so the database will have multiple copies of this user on persistent volumes).

### Architecture

Each environment includes a frontal server to serve static contents and pass API call to the backend, a backend server to query the database and serve API calls, and a database.

In each environment there are subtle changes, to account for the desired use of the environment (development, testing, etc.)

### Development

- The react frontend application is served with Node.

- The Express backend uses Nodemon, which reloads on code change (usefull for standalone llcal environments, in the developer's machine).

- The database container is statefull, as its data resides inside the container (it is destroyed whenever the container is removed).

### Staging

- The frontend is served with Nginx

- The backend is served with Node.

- Additional containers are used to build the static React and Express content of the client and server, and then pass it to the Nginx and Node containers respectively.

- The database data resides in a docker named volume (db_staging)

### Production

- Frontend and backend containers use the images built for staging.

- The database data resides in a docker named volume (db_prod)

- Multiple replicas of the front and backend (imutable) containers can be run with Swarm, to scale capacity according to demand.

![Environment containers and volumes](https://github.com/ptavaressilva/MERN_app_CI-CD_pipeline/blob/master/img/environments.png?raw=true)

## Running the project

Typically a developer wants a standalone development environment on her/his local machine, to develop and test the applicaation code.

Once code is ready, the developer pushes that code to the SCM repository (Gogs in our case), triggering the Continuous Delivery pipeline, and eventually releasing the code change to production.

![Jenkins pipeline](https://github.com/ptavaressilva/MERN_app_CI-CD_pipeline/blob/master/img/Jenkins_pipeline.png?raw=true)

### (Optional) Create a local development environment

To create a local development environment (on the developer's machine) the machine needs to have docker and docker-compose installed.

To create/start the development environment use `docker-compose -f ./app/docker-compose.dev.yml up -d`

The following folders on your computer will be mapped to frontend and backend server foldders:

| Folder on your computer | Server (container) | Folder on server   |
| ----------------------- | ------------------ | ------------------ |
| ./client/src            | Frontend           | /app/client/src    |
| ./client/public         | Frontend           | /app/client/public |
| ./server/src            | Backend            | /app/server/src    |

**Note:** when Jenkins builds and deploys the dev environment, the contents is copied to the container (not mapped).

You can then access the application on your local dev environment at [http://localhost:3000](http://localhost:3000)

### Step 1 - Create volumes for data persistance

Lets start by creating the docker volumes that will persist our pipeline and our staging and production database data:

`docker volume create ops_gogs ops_registry ops_jenkins app_db_staging cicd_stack_db_prod`

![Ops containers](https://github.com/ptavaressilva/MERN_app_CI-CD_pipeline/blob/master/img/Orchestration_and_volumes.png?raw=true)

### Step 2 - Configure Gogs

On startup, Gogs is served at port 3000, which will be used for our dev frontend, so we will change the Gogs port to 9001.

Start Gogs with:

`docker-compose -f ./ops/docker-compose.gogs3000.yml up -d`

Use `docker ps` to determine when the container with the image *gogs/gogs* is running, the connect to the Gogs webpage on [http://localhost:3000](http://localhost:3000) and configure the following parameters in the *Install Steps For First-time Run* page:

| Parameter                                                | Value                    |
| -------------------------------------------------------- | ------------------------ |
| Database Type                                            | SQLite3                  |
| HTTP Port                                                | 9001                     |
| Application URL                                          | `http://localhost:9001`/ |
| Admin Account Settings / Username                        | gogsadmin                |
| Admin Account Settings / Password (and confirm password) | gogssecret               |
| Admin Account Settings / email                           | a@a.a                    |


Hit the *"Install Gogs"* button to finish the configuration. **It will will not find the page, but that's ok**. The container needs to restart to pick up the port change. For now we are done with Gogs on port 3000, so we shut it down with:

`docker-compose -f ./ops/docker-compose.gogs3000.yml down`

Don't worry! Your configuration in saved in the `gogs` docker volume we created earlier.

### Step 3 - start Jenkins, Registry and Gogs

Start Jenkins, Docker Registry and Gogs (now on port 9001):

`docker-compose -f ./ops/docker-compose.ops.yml up -d`

#### Configure Jenkins

Use the `docker logs $(docker ps --filter name=ops_jenkins_1 -q)` command to view the Jenkins service logs and find the installation password, in a segment looking like this:

```bach
cicd_stack_jenkins.1.jft378dfo4tm@docker-desktop    | *************************************************************
cicd_stack_jenkins.1.jft378dfo4tm@docker-desktop    | *************************************************************
cicd_stack_jenkins.1.jft378dfo4tm@docker-desktop    | *************************************************************
cicd_stack_jenkins.1.jft378dfo4tm@docker-desktop    |
cicd_stack_jenkins.1.jft378dfo4tm@docker-desktop    | Jenkins initial setup is required. An admin user has been created and a password generated.
cicd_stack_jenkins.1.jft378dfo4tm@docker-desktop    | Please use the following password to proceed to installation:
cicd_stack_jenkins.1.jft378dfo4tm@docker-desktop    |
cicd_stack_jenkins.1.jft378dfo4tm@docker-desktop    | 18b9792120c6466f82f37c95363bc7bf
cicd_stack_jenkins.1.jft378dfo4tm@docker-desktop    |
cicd_stack_jenkins.1.jft378dfo4tm@docker-desktop    | This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
cicd_stack_jenkins.1.jft378dfo4tm@docker-desktop    |
cicd_stack_jenkins.1.jft378dfo4tm@docker-desktop    | *************************************************************
cicd_stack_jenkins.1.jft378dfo4tm@docker-desktop    | *************************************************************
cicd_stack_jenkins.1.jft378dfo4tm@docker-desktop    | *************************************************************
```

Open Jenkins (at [localhost:8000](http://localhost:8000/)), provide the password (in the capture above it is `18b9792120c6466f82f37c95363bc7bf`) and then click on the *Install suggested plugins* option.

After the pluggins finish installing, in the *Create First Admin User* page, create the following user:

| Field          | Value          |
| -------------- | -------------- |
| Username:      | pipelineadmin  |
| password:      | pipelinesecret |
| email address: | a@a.a          |

In the *Instance Configuration* page leave the URL as `http://localhost:8000/` and press the *Save and finish* button.

#### Optional: install Blue Ocean plugin

To install the additional plugins needed, go to *Manage Jenkins* (left-hand menu) > *Manage Plugins*, then click on the *Available* tab.

Type `Blue Ocean`on the text box and check the box for the pluggin with that name, then press *Download and install after restart*.

Finally, check the *Restart Jenkins when installation is complete and no jobs are running* checkbox at the bottom of the page.

### Step 4 - Create and clone the remote mern_app repository

Go to <http://localhost:9001/gogsadmin> and create a Gogs repository using the + dropdown on the top right corner. Call it `mern_app` and hit the *Create Repository* button.

To use the pipeline you need to clone the Gogs `mern_app` repository created above into a folder in your computer that must reside outside the folder where you cloned this [MERN_app_CI-CD_pipeline](https://github.com/ptavaressilva/MERN_app_CI-CD_pipeline) project.

```bash
cd ..
# you should be otside the MERN_app_CI-CD_pipeline folder
git clone http://localhost:9001/gogsadmin/mern_app
cd mern_app
```

As the Gogs repository is empty, you will get a warning, but it's ok.

```bash
warning: You appear to have cloned an empty repository.
```

Copy the contents of the [MERN_app_CI-CD_pipeline](https://github.com/ptavaressilva/MERN_app_CI-CD_pipeline) to the *mern_app* folder you created in [Clone the remote mern_app repository](#anchors-in-markdown) step, but **be carefull not to copy the  hidden .git folder** (if you copy from a local clone).

### Step 5 - Configure the pipeline

Go to Jenkins [http://localhost:8000](https://localhost:8000) and click *New Item* on the lefthand side menu. In the *Enter an item name* box, type `MERN app Continuous Delivery`, choose type *Multibranch Pipeline* and click *ok* at the bottom of the page.

In the next page, in the *General* tab, provide the values below:


| Field              | Value                                     |
| ------------------ | ----------------------------------------- |
| Branch source      | Git                                       |
| Project Repository | `http://gogs:9001/gogsadmin/mern_app.git` |
| Credentials > Add  | MERN app Continuous Delivery              |

In the *Folder Credentials Provider: MERN app Continuous Delivery* pop-up, provide the following values:

| Field    | Valiue       |
| -------- | ------------ |
| Username | `gogsadmin`  |
| Password | `gogssecret` |

Press the *Add* button to return to the pipeline *General* tab.

In the *Credetials* field, select gogsadmin/****

In the *Build Configuration* tab, leave *by Jenkisfile* selected and insert `ops/Jenkinsfile` in the *Script Path* box.

Below, in the *Scan Multibranch Pipeline Triggers* section, check *"Periodically if not otherwise run"*, set the interval to `1 minute` and press the *Save* button at the bottom of the page.

### Step 6 - Store development, staging and production environment variables

To store your development environment variables, create a file called `.env` in the ./app/ folder, with the following contents:

```bash
SRV_PORT=4000
MONGO_URI=db:27017/db?authSource=admin
MONGO_PORT=27017
MONGO_INITDB_ROOT_USERNAME=pedro
MONGO_INITDB_ROOT_PASSWORD=kewlSecret
NODE_ENV=development
```

*.env* files are excluded in gitignore, so they will not be pushed to the Gogs repository.

For the staging environment we will use Jenkins Credentials. To do this, go to [localhost:8000](http://localhost:8000/) and click *Manage Jenkins* on the lefthand side menu, then click on the *Manage crededntials* item. Next, click on *Jenkins* in the *Stores scoped to Jenkins* section and on *Global credentials (unrestricted)*.

Click *Add Crededntials* and repeat the proccess for each entry in the table below, using **Kind = Secret text**.

| Username                   | Password                          |
| -------------------------- | --------------------------------- |
| SRV_PORT                   | 4000                              |
| MONGO_URI                  | prod_db:27017/db?authSource=admin |
| MONGO_PORT                 | 27017                             |
| MONGO_INITDB_ROOT_USERNAME | produser                          |
| MONGO_INITDB_ROOT_PASSWORD | kewlSecret                        |
| NODE_ENV                   | production                        |

Finally, to store your production environment variables, create a file called `.env_prod` in the ./scripts/ folder, with the following contents:

```bash
SRV_PORT=4000
MONGO_URI=db:27017/db?authSource=admin
MONGO_PORT=27017
MONGO_INITDB_ROOT_USERNAME=pedro
MONGO_INITDB_ROOT_PASSWORD=kewlSecret
NODE_ENV=production
```

### Step 7 - Push code to run the pipeline

Now go to the *mern_app* folder,  make your first commit and push to Gogs:

```bash
git add .
git commit -m"Initial commit"
git push origin master
```

When prompted, provide `gogsadmin` as username and `gogssecret` as password.

Now you can go to the Jenkins page on [http://localhost:8000](https://localhost:8000) and watch the pipeline execute. If you installed the Blue Ocean plugin, click on *Blue Ocean* on the lefthand side menu to use this enhanced UI.

Once your code passes unit, integration and smoke tests, your commit will be deployed into production with a rolling update.
