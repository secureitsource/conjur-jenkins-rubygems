# Policy Demo for CyberArk Conjur & Jenkins

This is a demonstration of how to test and publish a RubyGem to the public RubyGems repository using CyberArk Conjur for Secrets Management and Jenkins for CI/CD.

## Policy File Breakdown

### rubygems.yml

[View rubygems.yml](1_rubygems.yml)

This policy creates three (3) secret variables to handle authentication to RubyGems via API - `username`, `password`, and `API key`.

```yaml
    - &variables
      - !variable username
      - !variable password
      - !variable api-key
```

This policy also creates a user group, called `api-users`, and allows them to read and execute the secret variables established within.

```yaml
    - !group
      id: api-users
      annotations:
        description: Users allowed to use the RubyGems API to publish/yank gems.

    - !permit
      role: !group api-users
      privileges: [ read, execute ]
      resources: *variables
```

### jenkins.yml

[View jenkins.yml](2_jenkins.yml)

This policy is for the Jenkins agents that will be testing the RubyGem and publishing it to the public RubyGems repository.

An `executors` layer and associated `host-factory` is established for accepting Jenkins agents that will be pulling source code from GitHub and testing our RubyGem.

```yaml
- !layer &executors-layer
    id: executors
    annotations:
      description: Layer for Jenkins executor hosts

  - !host-factory
    id: executors
    annotations:
      description: Host Factory for Jenkins executor hosts
    layer: [ *executors-layer ]
```

A `releasers` layer and associated `host-factory` is established for accepting Jenkins agents that will be publishing the final RubyGem to the public RubyGems repository.

```yaml
- !layer &releasers-layer
    id: releasers
    annotations:
      description: Layer for Jenkins releaser hosts

  - !host-factory
    id: releasers
    annotations:
      description: Host Factory for Jenkins releaser hosts
    layer: [ *releasers-layer ]
```

Last, we'll create one (1) secret variable for our private SSH key for cloning GitHub repos and permit the `secrets-users` group to read and execute it.  This will allow any member of the `secrets-users` group to be able to clone from GitHub as the `github_user: conjur-jenkins`.

```yaml
- &variables
    - !variable
      id: private-key
      annotations:
        description: Private SSH key for cloning GitHub repos
        github_user: conjur-jenkins

  - !group secrets-users
  - !permit
    role: !group secrets-users
    privileges: [ read, execute ]
    resources: *variables
```

### entitlements.yml

[View entitlements.yml](3_entitlements.yml)

The final policy that gets pushed to CyberArk Conjur is the one that outlines all the entitlements for our different publishing services.  In this demonstration, we'll need to be granted to two (2) roles: 1 for RubyGems API access and 1 for GitHub SSH access via `secrets-users`.

We'll be adding the `team-leads` user group and `ci/jenkins/releasers` layer to both RubyGems and GitHub grants.  While our `ci/jenkins/executors` only get access to our GitHub grant for cloning and testing the source code.

```yaml
# RubyGems grants
- !grant
  role: !group rubygems/api-users
  members:
    - !group team-leads
    - !layer ci/jenkins/releasers

# GitHub grants
- !grant
  role: !group github/secrets-users
  members:
    - !group team-leads
    - !layer ci/jenkins/executors
    - !layer ci/jenkins/releasers
```

### Jenkinsfile

[View Jenkinsfile](Jenkinsfile)

We build a pipeline and begin the `Test` stage first.  As you can see from the `agent label`, we'll be using our `executor-v2` which will have an established identity in our `ci/jenkins/executors` layer within Conjur.  At CyberArk, we create each stage in a shell script and wrap it in the Groovy DSL.  In this instance, we're running our `test.sh` shell script and then exporting results from `junit`.

```groovy
  stages {
    stage('Test') {
      steps {
        milestone(1)
        sh './test.sh'

        junit 'spec/reports/*.xml'
        junit 'features/reports/*.xml'
      }
    }
```

The next stage is our `Publish` stage.  Again, you can see from the `agent label`, we'll be using our `releaser-v2` this time.  We do a quick check on the `Test` stage for pass/failure.  If it passed, we merge the branch into `master` and start a 5-minute clock.  If it failed, back to the ol' drawing board!

```groovy
pipeline {
  agent { label 'executor-v2' }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  stages {
    stage('Test') {
      steps {
        milestone(1)
        sh './test.sh'

        junit 'spec/reports/*.xml'
        junit 'features/reports/*.xml'
      }
    }
```

Let's say it passed, that 5-minute clock started asking the question, "Publish to RubyGems?"  A team member now has 5 minutes on the clock to review the `junit` reports and decide whether to publish or not.  If "Yes" is chosen, we just run a `publish.sh` script and that publishes the RubyGem to the public RubyGems repository.

```groovy
// Only publish to RubyGems if branch is 'master'
    // AND someone confirms this stage within 5 minutes
    stage('Publish to RubyGems?') {
      agent { label 'releaser-v2' }

      when {
        allOf {
          branch 'master'
          expression {
            boolean publish = false

            if (env.PUBLISH_GEM == "true") {
                return true
            }

            try {
              timeout(time: 5, unit: 'MINUTES') {
                input(message: 'Publish to RubyGems?')
                publish = true
              }
            } catch (final ignore) {
              publish = false
            }

            return publish
          }
        }
      }
      steps {
        // Clean up first
        sh 'docker run -i --rm -v $PWD:/src -w /src alpine/git clean -fxd'

        sh './publish.sh'

        // Clean up again...
        sh 'docker run -i --rm -v $PWD:/src -w /src alpine/git clean -fxd'
        deleteDir()
      }
    }
  }
```