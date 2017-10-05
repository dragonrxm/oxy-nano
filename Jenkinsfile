def fail(reason) {
  def pr_branch = ''
  if (env.CHANGE_BRANCH != null) {
    pr_branch = " (${env.CHANGE_BRANCH})"
  }
  slackSend color: 'danger', message: "Build #${env.BUILD_NUMBER} of <${env.BUILD_URL}|${env.JOB_NAME}>${pr_branch} failed (<${env.BUILD_URL}/console|console>, <${env.BUILD_URL}/changes|changes>)\nCause: ${reason}", channel: '#lisk-nano-jenkins'
  currentBuild.result = 'FAILURE'
  sh 'rm -rf "$WORKSPACE/node_modules/"'
  sh 'N=${EXECUTOR_NUMBER:-0}; rm -rf ~/lisk-Linux-x86_64_$N'
  milestone 1
  error("${reason}")
}

node('lisk-nano-01'){
  stage ('Cleanup, Checkout, and Start Lisk Core') {
    try {
      deleteDir()
      checkout scm
    } catch (err) {
      fail('Stopping build, Checkout failed')
    }

    try {
      sh '''
	N=${EXECUTOR_NUMBER:-0}
	cp -r ~/lisk-Linux-x86_64 ~/lisk-Linux-x86_64_$N
	cd ~/lisk-Linux-x86_64_$N
	sed -i -r -e "s/^(.*ort\\":) 4000,/\\1 400$N,/" config.json
	sed -i -r -e "s/^(\\s*\\"port\\": 543)2,/\\1$N,/" config.json
	sed -i -r -e "s/^#(port\\s*=\\s*).*/\\1 543$N/" ~/lisk-Linux-x86_64_$N/pgsql/data/postgresql.conf
	export PGPORT=543$N
	bash lisk.sh rebuild -f ~/lisk-Linux-x86_64_$N/blockchain_explorer.db.gz
	'''
    } catch (err) {
      fail('Stopping build, Lisk Core failed to start')
    }
  }

  stage ('Install npm dependencies') {
    try {
      sh '''
      npm install
      # Build nano
      cd $WORKSPACE
      npm install
      '''
    } catch (err) {
      fail('Stopping build, npm install failed')
    }
  }

  stage ('Run Eslint') {
    try {
      ansiColor('xterm') {
	sh '''
	cd $WORKSPACE
	npm run eslint
	'''
      }
    } catch (err) {
      fail('Stopping build, Eslint failed')
    }
  }

  stage ('Build Nano') {
    try {
      sh '''
      # Add coveralls config file
      cp ~/.coveralls.yml-nano .coveralls.yml

      # Run Build
      npm run build
      '''
    } catch (err) {
      fail('Stopping build, Nano build failed')
    }
  }

  stage ('Run Unit Tests') {
    try {
      ansiColor('xterm') {
	sh '''
	export ON_JENKINS=true

	# Run test
	cd $WORKSPACE
	npm run test

	# Submit coverage to coveralls
	cat coverage/*/lcov.info | coveralls -v
	'''
    }
    } catch (err) {
      fail('Stopping build, Test suite failed')
    }
  }

  stage ('Start Dev Server and Run E2E Tests') {
    try {
      ansiColor('xterm') {
	sh '''
	N=${EXECUTOR_NUMBER:-0}
	export PGPORT=543$N
	# Run Dev build and Build
	cd $WORKSPACE
	export NODE_ENV=
	npm run dev -- --port 808$N &> .lisk-nano.log &
	sleep 30

	# End to End test configuration
	export DISPLAY=":1$N"
	Xvfb ":1$N" -ac -screen 0 1280x1024x24 &
	./node_modules/protractor/bin/webdriver-manager update

	# Run End to End Tests
	npm run e2e-test -- --params.baseURL "http://localhost:808$N/" --params.liskCoreUrl "http://localhost:400$N"

	cd "~/lisk-Linux-x86_64_$N"
	bash lisk.sh stop_node
	pkill -f Xvfb -9 || true
	rm -rf "/tmp/.X1$N-lock" || true
	pkill -f webpack -9 || true
	'''
      }
    } catch (err) {
      fail('Stopping build, End to End Test suite failed')
    }
  }

  stage ('Deploy and Set milestone') {
    try {
      sh '''
      rsync -axl --delete --rsync-path="mkdir -p '/var/www/test/lisk-nano/$BRANCH_NAME/' && rsync" "$WORKSPACE/app/dist/" "jenkins@master-01:/var/www/test/lisk-nano/$BRANCH_NAME/"

      # Cleanup - delete all files on success
      rm -rf "$WORKSPACE/*"
      '''
    } catch (err) {
      fail('Stopping build, Deploy failed')
    }
    milestone 1
    /* notify of success if previous build failed */
    previous_build = currentBuild.getPreviousBuild()
    if (previous_build != null && previous_build.result == 'FAILURE') {
      def pr_branch = ''
      if (env.CHANGE_BRANCH != null) {
	pr_branch = " (${env.CHANGE_BRANCH})"
      }
      slackSend color: 'good', message: "Recovery: build #${env.BUILD_NUMBER} of <${env.BUILD_URL}|${env.JOB_NAME}>${pr_branch} was successful.", channel: '#lisk-nano-jenkins'
    }
  }
}
