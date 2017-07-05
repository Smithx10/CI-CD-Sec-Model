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
* Groups:  Groups are Top Level Namespaces created from the **team-name** provided by the Project Manager.  They house all the build projects for a particular team.  ex.  /demo-team/com.rxcorp.ss.webapp\_art\_build
    * Access Levels:  Each group has the following 5 AccessLevels:
      * Owner
      * Master
      * Developer
      * Reporter
      * Guest
* Projects:  Projects inherit RBAC from their parent Groups.  Projects can also have local security settings.  Projects normally map 1 to 1 with their corresponding Jenkins Jobs. Currently there is **no official process** in regards to obtaining a project.  How do we educate about the following types of projects.
  * Build Projects:
    * Owned by the developers
    * Only Produce artifacts that are published into their corresponding Nexus namespace.
  * Deploy Projects
    * Owned by Operators and can be changed via a GitLab Merge Request
  * Misc Projects
    * Projects that are used for other purposes that are not apart of the build / deploy pipelines.  ex. Documentation.


### Nexus
* Blob Stores: Blob Stores are areas on disk allocated for streaming of the objects posted to nexus over http.  These stores have been corrupted in the passed.  In order to minimize our fault domains,  we should group or data appropriately. Blob Stores are consumed by Repositories / Regstires and are RBAC is not applied at this level.  We have a few options currently.
  * Allocate a Blob Store per Team-Name.  For non promoted Artifacts.  Promoted Artifacts should reside on a gated Blob Store or maybe even safer yet, separate spindles.
  * Allocate a Blob store per each the Dev, and Prod Environments.  This simplicity comes with a negative, in the event of failure, the failure domain is larger, possibly effecting all objects per store.


* Repositories / Registries: Repositories and Registries in Nexus differ depending on the client consuming them, but will share Blob stores if configured so.  The indexing of objects of objects is handled by the elastic search library.   Users interface at the Repository/Registry level and as Operators we can apply RBAC at this level along with the WebUI. Nexus has the ability to trigger events per repository,  if namespaced properly, these repository / registry events can be used to trigger downstream actions ( Think Stackstorm, Jenkins, "http -X POST" ).  I am biased, and believe in less complexity leads to less failures,  and believe we should be using the raw namespace and use simple unix tools to keep the model simpler, instead of attaching useless metadata to our objects.
  * Docker Registries:
    * Giving each team their own Isolated Docker registry will most likely not be possible due do the fact that we would need to update the docker.tar.gz on every mesosphere agent everytime we update / add a team.  (Built into triton lawlbladez, /slapsknee)  We could probably pull this off, but will require further planning.  For example, use pass the URI as a runtime variable so that each team can only access their registry.
    *  Must make a choice on this.  Dev Registry and Prod Registry only?
  * NPM Registry:
    * NPM registries can be name spaced per team.
  * Maven/Scala Repository:
    * Maven Registries I believe can be name spaced per team.
  * RAW:
    * Is the simplest and the most adaptable, build + tar.gz + post.  No dependencies on SBT/NPM/Maven for publishing, built your fully independent artifact and post it to the disk.  Simple = FRIEND for publishing or Maven,  pretty much it's going to work, because .... it's http, and doesn't change every time some gets the desire to add some "Features". Doesn't apply to docker registries.  1 Raw Repo requires less operating overhead, but requires the developer to understand how to use basic unix tools.
  * RBAC: Role based access contorl in Nexus allows you to create a Role and add registries / repositories under said Role.  If we want to support all varients, we will need to map the Team-Name (AD Group) to it's Role in Nexus. Scenario: Demo-Team Requires NPM (*nodejs*), Maven(*scala*, *java*), and Raw (*raw artifacts ex. JDK version.tar.gz*).
    1. Map Team-Name to Role: ADGroup: **INTERNAL\Demo-Developers** -> Demo-Team-Artifacts
    2. Nest **Demo-Team-NPM**, **Demo-Team-Maven**, **Demo-Team-RAW**, Under the **Demo-Team-Artifacts** Role.
    3. Nest the Proper WebUI Roles/Permissions also on the **Demo-Team-Artifacts** Role.


### DC/OS Marathon Services
*  DC/OS Marathon Services: This namespace should clone the deploy jobs namespace except we can drop the *type* from the name and only have the project name. I'd actually prefer to keep the environment in the service, to re-enforce where you are when working with a given service.  This should minimize human  operator error and confusion.
  * /**env**/**team-name**/**project-name**/**env**
  * /devl/team-name/project-name_devl
  * /sit/team-name/project-name_sit
  * /uacc/team-name/project-name_uacc
  * /prod/team-name/project-name_prod


### DC/OS Secrets
* DC/OS Secrets: This namespace should clone and the Marathon Services and Jenkins Deploy namespace for a few reasons.
  * Marathon Services can natively access their corresponding Secrets namespace.
    * ex. /foo/bar/baz can access /foo/bar/baz
  * The Security Domain for each is the same across all 3 services.  This means that the same group of users should have access to each of the 3 namespaces.
  * User Credential's should not have access to modify SIT,UACC,PROD namespaces and this action should only be reserved for the Deployment Services account that are stored in Jenkins.

### NFS
*  NFS Folder Structure:  The NFS Folder Structure should replicate the DC/OS Marathon Service Namespace.   This will allow an operator to find the data, and allow for easier Gitlab Merge Request review.  *ex. A marathon.json volume mapping should not be accessing different levels / namespaces on the NFS Mount.  **Marathon**: /devl/foo_service/frontend -> **NFS**: /uacc/bar_service/backend*
This is wrong for 3 reasons, **devl** is trying tom map **uacc** data, and the **foo** service is trying to access the **bar** service, and the **frontend** service is trying to access data from the **backend** service.  I cannot stress enough that this needs to be peer reviewed, or we will shoot our own foot off.

>Disclaimer: I, Bruce Smith, disagree with my upmost ability about using NFS for all of our storage.  This will only lead to disaster.  We have already seen an 18 Hour Outage due to a User Error Network Partition.   Shared Storage has a place... maybe... but for any distributed service depending on a "centralized, network storage solution" is not really Distributed now is it?  Data replication and high availability should be handle in the application tier  and not by magical block stores.

### On-Boarding LifeCycle for a Project
* Initial Information Required:
  * Team-Name (*This will be reused everywhere*)
    * **AD Group for Jenkins, Nexus, *DC/OS doesn't support groups in 1.9***.
  * Will the team be Publishing Artifacts or Deploying Services?
    * **AD Service Account for Publishing** into Nexus. This Credential as described are attached to the teams' Build Jenkins Folder. *ex. /**demo-team**/my_artifact_art_build*
    * **AD Service Accounts for Deploying** to each Environment.  These Credential's as described are attached to the team's Deployment Jenkins Folders. *ex. /devl_deploy/**demo-team**/my.service_devl_deploy*

* Operator Actions:
   * The DevOps Team will provision the following:
    * The Appropriate Groups in **GitLab**, along with the the appropriate users memberships.
    * The Base Seed-Job File for the teams to Pull Request against to create / modify their **Jenkins** Jobs. (Requires Merge Request)
    * Add the Deployment Credentials to the Appropriate Deployment Jenkins Job Folders. *ex. /sit_deploy/demo-team*
    * The Appropriate Repositories and Registries in **Nexus**.
    * ** TakeMerge Requests** after Developers have tested / explored their code in the Sandbox Cluster.

* Developer Actions:
  * Explore the Jobs-DSL plugin API, and submit Merge Requests after testing in the sandbox.
  * Explore the suite of Jenkins Pipeline Jobs in the sandbox and  Enjoy your new found self service.
  * Don't create 15 million gitlab repositories, unless you really have to.
  * Don't make the sandbox cluster, your production cluster.
  * When you have a question, ask, everyone is and has been new at something.
