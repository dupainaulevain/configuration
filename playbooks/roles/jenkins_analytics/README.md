Jenkins Analytics
=================

A role that sets-up Jenkins for deploying analytics stack
on AWS EMR.

It does following steps:

* Installs jenkins using ``jenkins_master``
* Configures ``config.xml`` to enable security and use
  Linux Auth Domain.
* Enable the use of jenkins clis

Configuration
-------------

Following variables are used by this role:

You need to provide a json file containing credentials
to be uploaded to server. This file is **a local file**
suitable for use in ``copy`` module. This file is set by
following variable:

    jenkins_credentials_file:  files/ansible-jenkins-credentials.json


Used by command waiting on jenkins start-up after running ``jenkins_master``
role:

    jenkins_connection_retries: 60
    jenkins_connection_delay: 0.5

Jenkins auth realm, that is a method used by jenkins to authenticate users.
Realm type stored in ``jenkins_auth_realm.name`` variable.

In future we will try enable other auth domains, while
preserving ability to run cli.

### Unix Realm

For now only ``unix`` realm supported --- which requires every user to have a
shell account on the server.

Unix realm requires following settings:

``service`` --- jenkins uses PAM configuration for this service. `su` is
a safe choice as it doesn't require that user has ability to login remotely/

``plain_password`` --- plaintext password, **you should change** default values

``crypted_password`` --- encrypted password, to obtain it use ``mkpasswd``
command, for example: `` mkpasswd --method=sha-512``

Example realm configuration:

    jenkins_auth_realm:
      name: unix
      service: su
      plain_password: jenkins
      crypted_password: $6$rAVyI.p2wXVDKk5w$y0G1MQehmHtvaPgdtbrnvAsBqYQ99g939vxrdLXtPQCh/e7GJVwbnqIKZpve8EcMLTtq.7sZwTBYV9Tdjgf1k.


### Seed job configuration

Seed job is configured in ``jenkins_seed_job`` variable, which has following
attributes:

``name`` Name of the job in Jenkins.

``time_trigger``: A jenkins cron entry defining how often this job is ran.

``removed_job_action`` what to do when job created by the seed job is deleted.
This can be either  ``DELETE`` or``IGNORE``.
``removed_view_action`` What to do when view created by the seed job is removed.
This can be either  ``DELETE`` or``IGNORE``.
``scm`` Scm object is used to define from where to download
the jobs. It has following properties:

``scm.type`` For today it must have value of ``git``.

``scm.url`` Url for the repository

``scm.credential_id`` Id of credential used to authenticate to the repository.

``scm.targed_jobs`` A shell glob expression relative to repo selecting jobs
to import

``scm.additional_classpath`` A path relative to repo root, pointing to a path
that contains additional groovy scripts used by the seed jons.

Example scm configuration:

    jenkins_seed_job:
      name: seed
      time_trigger: "H * * * *"
      removed_job_action: "DELETE"
      removed_view_action: "IGNORE"
      scm:
        type: git
        url: "git@github.com:edx-ops/edx-jenkins-job-dsl.git"
        credential_id: "github-deploy-key"
        targed_jobs: "jobs/analytics-edx-jenkins.edx.org/*Jobs.groovy"
        additional_classpath: "src/main/groovy"

Credential file example
-----------------------

To generate a proper json credential file I strongly suggest:

1. Write the file in ``yaml`` format.
2. Convert it by using for example:

        cat file.yaml |  python -c "import yaml,json,sys; print(json.dumps(yaml.load(sys.stdin)))" > file.json


See ``jenkins_analytics/meta/ansible-jenkins-credentials.yaml`` a example
configuration file format.

Known issues
------------

1. SSH keys without password don't work.
2. This playbook should have been a module: ``_execute_ansible_cli.yaml``
3. We shouldn't delete seed job every time, just check if it is already on
 server and if it's the case change it.
4. Anonymous user has discover and get job permission, as without it
  ``get-job``, ``build <<job>>`` commands wouldn't work.
  Giving anonymous these permissions a workaround for
  transient Jenkins issue (reported [couple][1] [of][2] [times][3]).
5. We force unix authentication method --- that is every user that can login
  to Jenkins also needs to have a shell account on master.
6. Password creds were not tested yet :)

Dependencies
------------

- jenkins_master

[1]: https://issues.jenkins-ci.org/browse/JENKINS-12543
[2]: https://issues.jenkins-ci.org/browse/JENKINS-11024
[3]: https://issues.jenkins-ci.org/browse/JENKINS-22143
