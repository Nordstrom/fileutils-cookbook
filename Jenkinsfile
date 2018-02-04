#!/usr/bin/env groovy
// This is a groovy-based Jenkinsfile for building cookbooks in Jenkins
// See https://jenkins.io/doc/book/pipeline/jenkinsfile/ and http://groovy-lang.org/semantics.html

// The code is generated by Jetbridge, based on your jetbridge.yml file. See the template and contribute at
// https://git.nordstrom.net/projects/ENG/repos/jetbridge-plugin/browse/src/main/resources/plan-templates/cookbook-Jenkinsfile.template

// This particular file was generated for the ITS/fileutils repo in BitBucket
// Data: {repository=fileutils, project=ITS, config={plan=cookbook}}

// Where to upload when we're done (master branch only)
supermarket_url = 'https://chef-supermarket.nordstrom.net'
chef_username = 'oppjenk01'
chef_orgs = [] //filled in based on your jetbridge.yml's deploy: orgs list

stage('Build') {
  // get a build node with chefdk and git
  node ('git && chefdk') {
    // output the chefdk / ruby versions for debugging
    sh 'chef --version'
    sh 'chef exec ruby --version'

    // pull the latest version down
    checkout scm

    // let BitBucket know we're in progress on a build, using specific username/pw
    step([$class: 'StashNotifier', credentialsId: 'bitbucket-build-notifier'])

    // read the version from metadata.rb
    version = cookbook_version()
    if (version) {
        echo "Building ${version}..."
    }
    else {
        error("You need to specify a cookbook version in your metadata.rb file")
    }

    build_error = null // null is "falsey" in groovy

    try {
      // since the build node runds in cloud v1, we need to use a proxy to access the internet
      // but not for nordstrom.net servers and not for localhost (e.g. chefspec endpoints)
      withEnv(["HTTPS_PROXY=http://pbcld-proxy.nordstrom.net:3128", "NO_PROXY=nordstrom.net,127.0.0.1"]) {

        commandPrefix = 'chef exec '
        // if we have a Gemfile, use it
        if (fileExists('Gemfile')) {
          // bundle install the gems required to a local vendor path (instead of the default system location)
          sh 'chef exec bundle install --path vendor/bundle'
          commandPrefix = 'chef exec bundle exec '
        }

        // if we have a Rakefile, use it
        if (fileExists('Rakefile')) {
          sh "${commandPrefix}rake"
        }
        else {
          // otherwise, use a sane default
          sh "${commandPrefix}rubocop"
          sh "${commandPrefix}foodcritic ."
          sh "${commandPrefix}rspec"
        }
      }

      // if we get to this point with no errors, we have a successful build
      currentBuild.result = 'SUCCESS'
    } catch(err) {
      // any error above means we have a failed build
      build_error = err
      currentBuild.result = 'FAILED'
    }

    // report status back to BitBucket
    step([$class: 'StashNotifier', credentialsId: 'bitbucket-build-notifier'])

    if (build_error) {
      error(build_error.toString)
    }
  }
}

// Only continue if we are on the master branch
if (env.BRANCH_NAME == 'master') {
  stage('Publish') {
    node ('git && chefdk') {
      version = cookbook_version()

      // check to see if the tag has already been made
      // if so, we want to bail early with a clear message
      if (sh(script: 'git tag', returnStdout: true).contains("cv${version}")) {
        error "There's already a ${version} version of this cookbook. \n" +
          "You'll need to bump the version (in metadata.rb) appropriately to get this to publish."
      }

      // tag this commit locally
      sh "git tag cv${version} -m 'Cookbook Version ${version}'"

      // push the tag back to BitBucket
      sshagent(['git_sdtcm01']) {
        sh "git push origin --tags"
      }

      echo "Publishing to Supermarket... ${version}"
      extra_switches=""

      // look at metadata.rb to see if we have a source/issues URL defined
      if (readFile('metadata.rb') =~ '^[[:space:]]*source_url') {
        if (readFile('metadata.rb') =~ '^[[:space:]]*issues_url') {
          extra_switches="--extended-metadata"
        }
      }

      // using the supermarket account for oppjenk01, upload the cookbook
      withCredentials([[$class: 'FileBinding', credentialsId: 'chef-supermarket', variable: 'CHEF_PRIVKEY']]) {
        sh "chef exec stove login --username ${chef_username} --key ~/.ssh/id_rsa"
        sh "chef exec stove \
        --no-git \
        --endpoint ${supermarket_url}/api/v1 \
        --username ${chef_username} \
        -k ${env.CHEF_PRIVKEY} \
        -l info \
        ${extra_switches}"
      }
    }
  }

  // upload the cookbook to appropriate orgs
  stage('ChefUpload') {
    node ('chefdk') {
      chef_org_upload(chef_orgs)
    }
  }
}

// find cookbook version from metadata.rb file
def cookbook_version() {
  def matcher = readFile('metadata.rb') =~ '[ \t]*version[ \t]+["\'](.+)["\']'
  matcher ? matcher[0][1] : null
}

def chef_org_upload(orgs) {
  // create Berksfile.lock if it doesn't exist, download deps
  sh "berks install"

  // have to use a traditional for loop - see https://gist.github.com/oifland/ab56226d5f0375103141b5fbd7807398
  for (int i = 0; i < orgs.size(); i++) {
    echo "Uploading to chef org ${orgs[i]}..."
    // use berks upload to send this cookbook and dependencies to the org
    sh "export chef_org=${orgs[i]} && chef exec berks upload"
  }
}
