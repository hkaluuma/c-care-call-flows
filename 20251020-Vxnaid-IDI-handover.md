# Codebase and Repository Ownership

The **Uganda Vaccination version of Vxnaid Mobile App** is avaiable under branch: https://github.com/ConnectForLife/vxnaid/tree/vrvu 

The repository has additional direct access configured for users:
* https://github.com/hkaluuma (maintain)
* https://github.com/Vinald (maintain)

The **Uganda Vaccination version of Vxnaid's CFL backend** is avaiable under branch: https://github.com/ConnectForLife/openmrs-distro-vxnaid/tree/vrvu 

* **No additional direct access rights configured.**

The Vxnaid source-code, and the CFL backend is stoted under Organization: https://github.com/connectforlife

The owners of the organization:
* https://github.com/druchniewicz
* https://github.com/jslawinski
* https://github.com/pgesek
* https://github.com/pwargulak

# Development Workflow

## CFL Backend

See: https://github.com/ConnectForLife/openmrs-distro-vxnaid/blob/vrvu/cfl/README.md

### Tests

Each module that a CFL backend consists of have own set of [unit](https://github.com/ConnectForLife/openmrs-module-callflows/tree/main/api/src/test) and [integration](https://github.com/ConnectForLife/openmrs-module-callflows/tree/main/omod/src/test) tests. Both kinds of tests are executed during regular build of the module. Standard Maven commands are used.

See Callflows readme as an example: https://github.com/ConnectForLife/openmrs-module-callflows/blob/main/README.md

## Vxnaid Mobile App

See Developer Guide: https://github.com/ConnectForLife/vxnaid/wiki/Architecture 

### Tests

The Vxnaid mobile app uses gradle to build, contains [unit](https://github.com/ConnectForLife/vxnaid/tree/main/app/src/test) and [integration](https://github.com/ConnectForLife/vxnaid/tree/main/app/src/androidTest) test. Both kinds of tests are executed during regular build of the Andorid app. Standard Gradle commands are used.

# System Architecture and Integrations

![Architecture Diagram](./Vxnaid-diagram-white.png)

There are no external systems.

1. Each Vxnaid Mobile App communicates directly with Custom CFL backend over REST.
2. Any communication between Vxnaid Mobile App and Custom CFL backend is initiated by the Mobile App.
3. Custom CFL backend uses AWS RDS to store the data.

## Connect for Life API

The OpenMRS follows convention of having API documentation deployed togather with the app.

The Vxnaid app uses APIs exposed by [Biometric Module](http://13.247.205.252/openmrs/ms/uiframework/resource/vxnaidbiometric/swagger/index.html) ([offline](https://github.com/ConnectForLife/openmrs-module-vxnaid-biometric/tree/main/omod/src/main/webapp/resources/swagger))

Core OpenMRS modules API can be seen [here](http://13.247.205.252/openmrs/module/webservices/rest/apiDocs.htm).

Some CFL modules expose own API documentation pages like, [Call flows](http://13.247.205.252/openmrs/ms/uiframework/resource/callflows/swagger/index.html) (for offline, build the module and inspect `target` directory).

For backup and restore see *Deployment and Hosting* section.

## Vxnaid Mobile App

Consumes the Connect for Life API.

## (Not included in current Uganda Vxnaid) Biometric server

The Vxnaid doesn't use biometric component, but the [Biometric Module](https://github.com/ConnectForLife/openmrs-module-vxnaid-biometric/blob/main/api/src/main/java/org/openmrs/module/biometric/api/service/BiometricService.java) uses Neurotechnology SDKs to communicate with the Biometric server.

# Database and Data Handling

There is no formal schema document fot the Connect for Life Backend or the Vxnaid Mobile app. 
The Connect for Life backend cosists of modules that each contribute own database model.
The App has single schema definition file [here](https://github.com/ConnectForLife/vxnaid/tree/8ea37ee80c98574f3d480582b756df15990f497d/app/schemas/com.jnj.vaccinetracker.common.data.database.ParticipantRoomDatabase), but it's more a source code rather then formal document.

## Connect for Life Backend

Migrations are automatically applied by each OpenMRS module, example of migration see [callFlow-2021-02-09-12:00](https://github.com/ConnectForLife/openmrs-module-callflows/blob/main/api/src/main/resources/liquibase.xml).

## Vxnaid Mobile app

Uses Room, automatic migrations generated based on [schema.json](https://github.com/ConnectForLife/vxnaid/tree/8ea37ee80c98574f3d480582b756df15990f497d/app/schemas/com.jnj.vaccinetracker.common.data.database.ParticipantRoomDatabase).

# Open Source Community and Licensing

There are no defined stadards regarding outside contribution to Vxnaid Mobile app, Vxnaid CFL, or base Connect for Life.

There is no active community, there is Connect for Life Slack channel within OpenMRS Slack: https://openmrs.slack.com/archives/C05A89D4WLX 

See *Codebase and Repository Ownership* section for ownership details, tha Slack is owned by OpenMRS, slack channel is open to anyone, created by pwargulak@soldevelo.com.

# Deployment and Hosting

**The IDI Vxnaid deployment is managed entirly via manual actions, there is no CD implemented.** 
The DevOps engineer is expected to directly connect to the server over SSH and directly interact with Docker. 

The Docker images are not published, see [README.md](https://github.com/ConnectForLife/openmrs-distro-vxnaid/blob/vrvu/cfl/README.md) for example of manual moving an image to production.

## Custom Connect for Life backend

Currently hosted in Docker running on EC2 13.247.205.252. The utility scripts are located in: `/home/ubuntu/cfl` directory. 

### Update process
1. Connect to EC2 13.247.205.252 over SSH.
2. Navigate to `/home/ubuntu/cfl/distroRepo/openmrs-distro-vxnaid/cfl`
3. Pull latest changes using Git Pull.
4. Build the image: `docker compose -f docker-compose.build.yml build web`
5. Navigate to `/home/ubuntu/cfl`
6. Copy new `docker-compose.run.yml` from `/home/ubuntu/cfl/distroRepo/openmrs-distro-vxnaid/cfl`
7. Optionally create a snapshot of database in AWS RDS - for rollback.
8. Run `./startDistro.sh`

#### Keep in mind:

1. The container doesn't store anything production-critical, it can be freely recreated.
2. The `/home/ubuntu/cfl/.env` contains secrets and DB connection details.
3. The `cfl_web-data` volume contains CallFlow configuration, this is critical for production but can be recreated easily as it is not a runtime-created data. (**There is no Backup, you must keep a copy of Callflows by yourself! Thay are not in DB!**)
4. The CFL uses AWS RDS database, that is up to you how often backups are created there. A common pattrern is do daily backups, that are stored for 7days, weekly and monthly backups stored for a quarter and a year respectively.
5. You can freely restart the `cfl_web_1` container, including `--force-recreate` option - the volume contains important data, not a container.

### Rollback process

*This was never actually executed.*

#### If there was no data model changes

1. Stop the `cfl_web_1` container.
2. The `cfl_web-data` volume contains OpenMRS/CFL modules at `/var/lib/docker/volumes/cfl_web-data/_data/modules`, this modules are loaded in version order, you must ensure there are no newer module versions. You can freely delete the contents of this directory. The startup process copies the modules into this directory everytime - it's recreated automatically.
3. Rerun *Update Process*, but choose previous version in `docker-compose.run.yml`.

#### (Quick, data loss possible) If there was a data model change

1. Stop the `cfl_web_1` container.
2. The `cfl_web-data` volume contains OpenMRS/CFL modules at `/var/lib/docker/volumes/cfl_web-data/_data/modules`, this modules are loaded in version order, you must ensure there are no newer module versions. You can freely delete the contents of this directory. The startup process copies the modules into this directory everytime - it's recreated automatically.
3. Use AWS RDS to rollback database to state from before an update. **Possible data loss.**
4. Follow *Update Process*, but choose previous version in `docker-compose.run.yml`.

#### (Slow, no data loss) If there was a data model change

1. Prepare the new, fixed version of the Vxnaid Connect for Life backend, thatb optionally performs rollback of data model changes.
2. Run *Update process*

### Downtime

There is no handling for downtime. The expectation is that each Vxnaid Mobile application can detect an offline server and delay the data push for later.

## Custom Vxnaid Mobile app

### Update process

It's expected that you will use MDM or manually distribute the app, and the update process comes down to installing the new version using regular Android app installation process. The automated migrations are applied to the app's internal DB. 

### Rollback process

**No process defined.**
The app uses internal database (Room) with automatic migrations generated from [schema](https://github.com/ConnectForLife/vxnaid/tree/8ea37ee80c98574f3d480582b756df15990f497d/app/schemas/com.jnj.vaccinetracker.common.data.database.ParticipantRoomDatabase). **There is no rollback implemented.**

It is expected that App moves always forward - so a new version that fixes problem is released, rather a rollback to previous one is made.

# Monitoring and Maintenance 

1. Daily - Manual monitoring of `cfl_web_` logs. Search for exceptions indicating a stuck/crashed app.
2. Daily - Manual monitoring of Web page availability.
3. As needed - Update OpenMRS modules with critical security fixes.
4. ? - Watch AWS RDS for end-of-support for the used MySQL version.

# General Handover and Knowledge Transfer

1. No automated deployment or CI.
2. No monitoring.
3. No out-of-the box solution for deployment of mobile app.
4. No build-in load balancing for mobile-to-backend communication.
4. It's a fork of Vxnaid.
5. **Keep an eye** for version names, any custom-made OpenMRS module has a `-vrvu.X` or `-cfl.X` suffix in the version name! They can be found in Connect for Life GitHub org, examples: [openmrs-module-cfl](https://github.com/ConnectForLife/openmrs-module-cfl/tree/vrvu), [openmrs-module-reporting-rest](https://github.com/ConnectForLife/openmrs-module-reportingrest/tree/1.14.0-cfl.X).

