def skip_snapshot = false
def did_deploy_artifact = false
def application_version = null

pipeline {
    agent {
        label 'docker'
    }

    parameters {
        booleanParam(
                name: "RELEASE",
                defaultValue: false,
                description: "Release a non-snapshot version"
        )
        booleanParam(
                name: "SNAPSHOT",
                defaultValue: false,
                description: "Release a snapshot version"
        )
        choice(
                name: "VERSION_INCREMENT",
                choices: ['Patch', 'Minor', 'Major'],
                description: "Which version part to increment (when releasing)"
        )
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        APPLICATION_VERSION = readMavenPom().getVersion()
        SLACK_CHANNEL = '#cc_refinery_alerts'
    }

    stages {
        stage('CheckSnapshot') {
            steps {
                script {
                    if (!params.RELEASE) {
                        setBuildName useMavenFormat: true
                    }
                    POM = readMavenPom file: 'pom.xml'
                    if (!(POM.version =~ /-SNAPSHOT$/)) {
                        skip_snapshot = true
                    }
                }
            }
        }
        stage('Clean') {
            steps {
                buildMaven goal: "clean"
            }
        }
        stage('Build') {
            steps {
                buildMaven goal: "package"
                step( [ $class: 'JacocoPublisher',
                        exclusionPattern: '**/com/fanthreesixty/sgs/tests/*'] )
            }
        }
        stage('Publish Snapshot') {
            when {
                not {
                    anyOf {
                        expression { return params.RELEASE }
                        expression { return skip_snapshot }
                    }
                }
                anyOf {
                    environment name: 'GIT_BRANCH', value: 'master'
                    environment name: 'GIT_BRANCH', value: 'origin/master'
                    expression { return  params.SNAPSHOT }
                }
            }
            steps {
                buildMaven goal: "deploy -Prelease -U -Dversion.buildNumber=${env.BUILD_NUMBER}"
                script {
                    did_deploy_artifact = true
                    application_version = env.APPLICATION_VERSION
                }
            }
        }
        stage('Publish Release') {
            when {
                expression { return params.RELEASE }
                anyOf {
                    environment name: 'GIT_BRANCH', value: 'master'
                    environment name: 'GIT_BRANCH', value: 'origin/master'
                }
            }
            steps {
                script {
                    scmCheckout()
                    POM = readMavenPom file: 'pom.xml'
                    def (major, minor, patch) = POM.version.replace('-SNAPSHOT', '').tokenize('.')
                    patch = patch ?: "0"

                    if (env.VERSION_INCREMENT == 'Patch') {
                        env.RELEASE_VERSION = major + '.' + minor + '.' + patch
                    } else if (env.VERSION_INCREMENT == 'Minor') {
                        env.RELEASE_VERSION = major + '.' + (minor.toInteger() + 1) + '.0'
                    } else if (env.VERSION_INCREMENT == 'Major') {
                        env.RELEASE_VERSION = (major.toInteger() + 1) + '.0.0'
                    } else {
                        error "Unexpected value of VERSION_INCREMENT: " + env.VERSION_INCREMENT
                    }
                }

                setBuildName name: env.RELEASE_VERSION

                buildMaven goal: "--batch-mode release:prepare -Dtag=v${env.RELEASE_VERSION} -DreleaseVersion=${env.RELEASE_VERSION}"
                buildMaven goal: "release:perform -DskipTests -Darguments=-DskipTests"
                script {
                    did_deploy_artifact = true
                    application_version = env.RELEASE_VERSION
                }
            }
        }
    }

    post {
        always {
            notifySlack channel: env.SLACK_CHANNEL
            logstashSend failBuild: false, maxLines: 0
        }
    }
}
