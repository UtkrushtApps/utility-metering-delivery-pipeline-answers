pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 30, unit: 'MINUTES')
  }

  environment {
    // Filled during "Detect changes"
    COMMIT_SHA = ''
    AFFECTED_SERVICES = '' // comma-separated
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        // Ensure we have enough history to diff against the previous commit/target.
        sh 'git fetch --no-tags --prune --unshallow || true'
      }
    }

    stage('Detect changes') {
      steps {
        script {
          def services = ['meter-ingest', 'usage-rating', 'billing-export', 'customer-alert']
          def serviceToTagEnv = [
            'meter-ingest': 'METER_INGEST_TAG',
            'usage-rating': 'USAGE_RATING_TAG',
            'billing-export': 'BILLING_EXPORT_TAG',
            'customer-alert': 'CUSTOMER_ALERT_TAG'
          ]

          def commitSha = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
          env.COMMIT_SHA = commitSha

          // Jenkins job is configured to run on main, but keep diff logic robust.
          def baseRef = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT?.trim()
          if (!baseRef) {
            baseRef = 'HEAD~1'
          }

          // If HEAD~1 doesn't exist (edge cases), fall back to comparing nothing.
          def baseSha = ''
          try {
            baseSha = sh(returnStdout: true, script: "git rev-parse ${baseRef}").trim()
          } catch (ignored) {
            baseSha = commitSha
          }

          def diffOut = sh(returnStdout: true, script: "git diff --name-only ${baseSha} ${commitSha} || true").trim()
          def changedFiles = diffOut ? diffOut.split('\n') : []

          if (changedFiles.isEmpty()) {
            currentBuild.description = "No relevant file changes"
            env.AFFECTED_SERVICES = ''
            return
          }

          boolean sharedChanged = changedFiles.any { it.startsWith('shared/') }

          def affected = [] as Set
          if (sharedChanged) {
            // Shared support package affects all services.
            affected.addAll(services)
          } else {
            for (String s : services) {
              def prefix = "services/${s}/"
              if (changedFiles.any { it.startsWith(prefix) }) {
                affected.add(s)
              }
            }
          }

          def affectedList = affected as List
          affectedList.sort()

          env.AFFECTED_SERVICES = affectedList.join(',')

          if (affectedList.isEmpty()) {
            currentBuild.description = "No service impacts detected (changed: ${changedFiles.take(10).join(',')})"
          } else {
            currentBuild.description = "Impacted services: ${affectedList.join(', ')} at ${commitSha.take(7)}"
          }

          // Persist for later stages.
          env.SERVICE_TO_TAG_ENV = serviceToTagEnv.collect { k, v -> "${k}=${v}" }.join(',')
        }
      }
    }

    stage('Build impacted services (parallel)') {
      when {
        expression { return env.AFFECTED_SERVICES?.trim() }
      }
      steps {
        script {
          def servicesToBuild = env.AFFECTED_SERVICES.tokenize(',')
          def parallelSteps = [:]

          for (String s : servicesToBuild) {
            parallelSteps["build:${s}"] = {
              sh "docker build -t ${s}:${env.COMMIT_SHA} -f services/${s}/Dockerfile ."
            }
          }

          parallel parallelSteps
        }
      }
    }

    stage('Deploy impacted services (main only, controlled)') {
      when {
        allOf {
          branch 'main'
          expression { return env.AFFECTED_SERVICES?.trim() }
        }
      }
      steps {
        script {
          def servicesToDeploy = env.AFFECTED_SERVICES.tokenize(',')

          def serviceToTagEnv = [:]
          if (env.SERVICE_TO_TAG_ENV?.trim()) {
            env.SERVICE_TO_TAG_ENV.split(',').each { pair ->
              def parts = pair.split('=', 2)
              serviceToTagEnv[parts[0]] = parts[1]
            }
          }

          withCredentials([string(credentialsId: 'sandbox-id', variable: 'SANDBOX_ID')]) {
            def targets = servicesToDeploy.join(', ')
            // Main-line-only controlled release gate.
            input message: "Deploy impacted services [${targets}] to sandbox '${SANDBOX_ID}' (commit ${env.COMMIT_SHA.take(7)})?", ok: 'Deploy'

            for (String s : servicesToDeploy) {
              def tagEnv = serviceToTagEnv[s]
              if (!tagEnv) {
                error("Missing tag env mapping for service '${s}'")
              }
              sh "python3 deploy_script.py --image ${s} --sha ${env.COMMIT_SHA} --compose-service ${s} --tag-env ${tagEnv}"
            }
          }
        }
      }
    }
  }
}
