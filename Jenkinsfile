/*
 * (C) Copyright 2020 Nuxeo (http://nuxeo.com/) and others.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 *
 * Contributors:
 *     Arnaud Kervern <akervern@nuxeo.com>
 *     Antoine Taillefer <ataillefer@nuxeo.com>
 *     Nelson silva <nsilva@nuxeo.com>
 */
 properties([
  [$class: 'GithubProjectProperty', projectUrlStr: 'https://github.com/nuxeo/jx-webui-builders/'],
  [$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', daysToKeepStr: '60', numToKeepStr: '60', artifactNumToKeepStr: '5']],
  disableConcurrentBuilds(),
])

void setGitHubBuildStatus(String context, String message, String state) {
  step([
    $class: 'GitHubCommitStatusSetter',
    reposSource: [$class: 'ManuallyEnteredRepositorySource', url: 'https://github.com/nuxeo/jx-webui-builders'],
    contextSource: [$class: 'ManuallyEnteredCommitContextSource', context: context],
    statusResultSource: [$class: 'ConditionalStatusResultSource', results: [[$class: 'AnyBuildResult', message: message, state: state]]],
  ])
}

void getReleaseVersion() {
  // git credentials required to invoke `jx-release-version` on a private repository
  sh """
    jx step git credentials
    git config credential.helper store
  """
  return sh(returnStdout: true, script: 'jx-release-version')
}

void setYumRepoCredentials() {
  dir('builder-java') {
    sh """
      envsubst < nuxeo.repo > nuxeo.repo~gen
      mv nuxeo.repo~gen nuxeo.repo
    """
  }
}

/**
 * Replaces environment variables present in the given yaml file and then runs skaffold build on it.
 * Needed environment variables are generally:
 * - DOCKER_REGISTRY
 * - ORG
 * - VERSION
 */
void skaffoldBuild(String yaml) {
  sh """
    envsubst < ${yaml} > ${yaml}~gen
    skaffold build -f ${yaml}~gen
  """
}

void skaffoldBuildRepository() {
  echo "Build and push builder Docker images using version ${VERSION}"
  skaffoldBuild('builder-maven-nodejs-chrome/skaffold.yaml')
}

pipeline {
  agent {
    label 'jenkins-jx-base'
  }
  environment {
    ORG = 'nuxeo'
  }
  stages {
    stage('Build and push builder images') {
      when {
        branch 'PR-*'
      }
      steps {
        setGitHubBuildStatus('build', 'Build and push builder images', 'PENDING')
        container('jx-base') {
          withEnv(["VERSION=${getReleaseVersion()}-${BRANCH_NAME}"]) {
            withCredentials([usernamePassword(credentialsId: 'packages.nuxeo.com-auth', usernameVariable: 'YUM_REPO_USERNAME', passwordVariable: 'YUM_REPO_PASSWORD')]) {
              setYumRepoCredentials()
              skaffoldBuildRepository()
            }
          }
        }
      }
      post {
        success {
          setGitHubBuildStatus('build', 'Build and push builder images', 'SUCCESS')
        }
        failure {
          setGitHubBuildStatus('build', 'Build and push builder images', 'FAILURE')
        }
      }
    }
    stage('Build and push release images') {
      when {
        branch 'master'
      }
      steps {
        setGitHubBuildStatus('release', 'Build and push release builder images', 'PENDING')
        container('jx-base') {
          withEnv(["VERSION=latest"]) {
            withCredentials([usernamePassword(credentialsId: 'packages.nuxeo.com-auth', usernameVariable: 'YUM_REPO_USERNAME', passwordVariable: 'YUM_REPO_PASSWORD')]) {
              setYumRepoCredentials()
              skaffoldBuildRepository()
            }
          }
        }
      }
      post {
        success {
          setGitHubBuildStatus('release', 'Build and push release builder images', 'SUCCESS')
        }
        failure {
          setGitHubBuildStatus('release', 'Build and push release builder images', 'FAILURE')
        }
      }
    }
    stage('GitHub release') {
      when {
        branch 'master'
      }
      steps {
        container('jx-base') {
          script {
            def currentNamespace = sh(returnStdout: true, script: "jx --batch-mode ns | cut -d\\' -f2").trim()
            if (currentNamespace == 'webui-staging') {
              echo "Running in namespace ${currentNamespace}, skip GitHub release stage."
              return
            }
            setGitHubBuildStatus('github-release', 'GitHub release', 'PENDING')
            withEnv(["VERSION=${getReleaseVersion()}"]) {
              echo "Releasing version ${VERSION}"
              sh """
                #!/usr/bin/env bash -xe

                # Git tag
                jx step tag -v ${VERSION}

                # Git release
                jx step changelog -v v${VERSION}
              """
            }
          }
        }
      }
      post {
        always {
          step([$class: 'JiraIssueUpdater', issueSelector: [$class: 'DefaultIssueSelector'], scm: scm])
        }
        success {
          setGitHubBuildStatus('github-release', 'GitHub release', 'SUCCESS')
        }
        failure {
          setGitHubBuildStatus('github-release', 'GitHub release', 'FAILURE')
        }
      }
    }
  }
}
