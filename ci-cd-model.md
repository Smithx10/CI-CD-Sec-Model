# CI-CD-Sec-Model
----


## On-Boarding Requirements

1. Team-Name: ex. **Demo-Team** . Team's require an AD User Group with members of the Development team to apply RBAC in Jenkins and Nexus.
  * ex. Jenkins Team Folder: **INTERNAL\Demo-Developers**
  * ex. Nexus Roles, which give access to Repositories/Registries : **INTERNAL\Demo-Developers**

2.  Project-Name: **com.rxcorp.ss.webapp\_art_build**.
  * Are Required to be of the following type.  
    * art_build: Used to build and publish an artifact into nexus. Artifact builds require a *Publish Credential*  which current is an AD Service account. ex. **INTERNAL\CDTASVC-JEN_DEMO-D**
    * docker_build: Used to build and publish an docker image into nexus.  Docker builds also require the need of this account.
    * **env**_deploy: Used to deploy an image / artifact into an environment.   **env** describes the environment you are deploying into and can be: *devl, sit, uacc, prod* . Deploy Jobs  require a service account per environment.




### Jenkins

* Build Jobs *(Publish Credential is Stored at team-name  folder)* . The *Publish Credential* is an AD Service account that can publish into Nexus.  ex: **INTERNAL\CDTASVC-JEN_DEMO-D**
  * /team-name/artefact\_art\_build
  * /team-name/artefact\_docker\_build

* Deploy Jobs *(Deploy Credential is Stored at team-name folder)* The **Deploy Credential** Is given access to the appropriate environment's namespace. ex. **INTERNAL\CDTASVC-JEN_DEMO-D** can only deploy to the /devl/demo-team namespace on DC/OS.

  * /devl\_deploy/**team-name**/project-name\_devl\_deploy

  * /sit\_deploy/**team-name**/project-name\_sit\_deploy

  * /uacc\_deploy/**team-name**/project-name\_uacc\_deploy

  * /prod\_deploy/**team-name**/project-name\_prod\_deploy

> *Note: Build Jobs can trigger downstream deploy Jobs which are gated by user input if desired.  The configuration for all of this is stored in the corresponding GIT deploy projects, which are maintained by the DevOps and ProdOps teams.  The use of Merge Requests allows a developer the ability to make deployment changes, which are peer reviewed.*

### Gitlab
* Groups:  Groups are Top Level Namespaces created from the **team-name** provided by the Project Manager.  The house all the build projects for a particular team.  ex.  /demo-team/com.rxcorp.ss.webapp\_art\_build    
    * Access Levels:  Each group has the following 5 AccessLevels:
      * Owner
      * Master
      * Developer
      * Reporter
      * Guest
* Projects:  Projects inherit RBAC from their parent Groups.  Projects can also have local security settings.  Projects normally map 1 to 1 with their corresponding Jenkins Jobs.  
  * Build Projects

  * Deploy Projects

  * Misc Projects


### Nexus
* Repositories

### DC/OS
*


### DC/OS Secrets  
*
