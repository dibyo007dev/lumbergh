@Library('github.com/mozmeao/jenkins-pipeline@20170315.1')
def stage_deployed = false
def config
def docker_image

conduit {
  node {
    stage("Prepare") {
      checkout scm
      setGitEnvironmentVariables()

      try {
        config = readYaml file: "jenkins.yml"
      }
      catch (e) {
        config = []
      }
      println "config ==> ${config}"

      if (!config || (config && config.pipeline && config.pipeline.enabled == false)) {
        println "Pipeline disabled."
      }
    }

    docker_image = "${config.project.docker_name}:${GIT_COMMIT_SHORT}"

    stage("Build") {
      if (!dockerImageExists(docker_image)) {
        sh "echo 'ENV GIT_SHA ${GIT_COMMIT}' >> Dockerfile"
        dockerImageBuild(docker_image, ["pull": true])
      }
      else {
        echo "Image ${docker_image} already exists."
      }
    }

    stage("Test") {
      parallel "Lint": {
        dockerRun(docker_image, "flake8 careers")
      },
      "Unit Test": {
        def db_name = "mariadb-${env.GIT_COMMIT_SHORT}-${BUILD_NUMBER}"
        def args = [
          "docker_args": ("--name ${db_name} " +
                          "-e MYSQL_ALLOW_EMPTY_PASSWORD=yes " +
                          "-e MYSQL_DATABASE=careers"),
          "cmd": "--character-set-server=utf8mb4 --collation-server=utf8mb4_bin",
          "bash_wrap": false
        ]

        dockerRun("mariadb:10.0", args) {
          args = [
            "docker_args": ("--link ${db_name}:db " +
                            "-e CHECK_PORT=3306 " +
                            "-e CHECK_HOST=db")
          ]
          dockerRun("giorgos/takis", args)

          args = [
            "docker_args": ("--link ${db_name}:db " +
                            "-e 'DEBUG=False' " +
                            "-e 'ALLOWED_HOSTS=*' " +
                            "-e 'SECRET_KEY=foo' " +
                            "-e 'DATABASE_URL=mysql://root@db/careers' " +
                            "-e 'SECURE_SSL_REDIRECT=False'"),
            "cmd": "coverage run ./manage.py test"
          ]
          dockerRun(docker_image, args)
        }
      }
    }

    stage("Upload Images") {
      dockerImagePush(docker_image, "mozjenkins-docker-hub")
    }
  }

  milestone()

  def deployStage = false
  def deployProd = false

  node {
    onBranch("master") {
      deployStage = true
    }
    onTag(/\d{4}\d{2}\d{2}.\d{1,2}/) {
      deployProd = true
    }
  }

  if (deployStage) {
    for (deploy in config.deploy.stage) {
      lock("push to ${deploy.name}") {
        stage ("Deploying to ${deploy.name}") {
          node {
            deis_executable = deploy.deis_executable ?: "deis"
            deisLogin(deploy.url, deploy.credentials, deis_executable) {
                deisPull(deploy.app, docker_image, null, deis_executable)
            }
            newRelicDeployment(deploy.newrelic_app, env.GIT_COMMIT_SHORT,
                               "jenkins", "newrelic-api-key")
          }
        }
        stage ("Acceptance tests against ${deploy.name}") {
          node {
            sh "tests/acceptance_tests.sh ${deploy.app_url}"
          }
        }
      }
    }
  }
  if (deployProd) {
    for (deploy in config.deploy.prod) {
      stage ("Deploying to ${deploy.name}") {
        node {
          lock("push to ${deploy.name}") {
            deis_executable = deploy.deis_executable ?: "deis"
            deisLogin(deploy.url, deploy.credentials, deis_executable) {
              deisPull(deploy.app, docker_image, null, deis_executable)
            }
            newRelicDeployment(deploy.newrelic_app, env.GIT_COMMIT_SHORT,
                               "jenkins", "newrelic-api-key")
          }
        }
      }
      stage ("Acceptance tests against ${deploy.name}") {
        node {
          lock("push to ${deploy.name}") {
            sh "tests/acceptance_tests.sh ${deploy.app_url}"
          }
        }
      }
    }
  }
}
