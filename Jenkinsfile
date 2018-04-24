#!groovy

// Don't test plugin compatibility - exceeds 1 hour timeout
// Allow failing tests to retry execution
// buildPlugin(failFast: false)

// Test plugin compatibility to latest Jenkins LTS
// Allow failing tests to retry execution
buildPlugin(jenkinsVersions: [null, '2.60.1'],
            findbugs: [run:true, archive:true, unstableTotalAll: '0'],
            failFast: false)

node("docker && highmem") {
    // TODO: this VM is not a single-shot one, we need to wipe it on our own
    dir(env.BUILD_NUMBER) {

        stage("Integration Tests: Checkout") {
            infra.checkout()
        }

        def mvnSettingsFile = "${pwd tmp: true}/settings-azure.xml"
        def mvnSettingsFileFound = infra.retrieveMavenSettingsFile(mvnSettingsFile)
        def outputWAR
        def metadataPath

        dir("src/test/it") {
            // TODO: convert to a library method
            def outputWARpattern = "target/custom-war-packager-maven-plugin/output/target/jenkins-remoting-it-1.0-remoting-it-SNAPSHOT.war"
            stage("Build Custom WAR") {
                List<String> mavenOptions = [
                        '--batch-mode', '--errors',
                        'clean', 'install',
                        "-Dcustom-war-packager.batchMode=true",
                        "-Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn"
                ]

                if (mvnSettingsFileFound) {
                    mavenOptions << "-s"
                    mavenOptions << "${mvnSettingsFile}"
                    mavenOptions << "-Dcustom-war-packager.mvnSettingsFile=${mvnSettingsFile}"
                }

                timeout(30) {
                    infra.runMaven(mavenOptions)
                }
                archiveArtifacts artifacts: outputWARpattern

                // Pass variables for the next steps
                outputWAR = pwd() + "/" + outputWARpattern
                metadataPath = pwd() + "/essentials.yml"
            }
        }

        def fileUri = "file://" + outputWAR
        stage("Run ATH") {
            dir("ath") {
                runATH jenkins: fileUri, metadataFile: metadataPath
                deleteDir()
            }
        }

        stage("Run PCT") {
            //TODO: Remove Slf4jMavenTransferListener option once runPCT() invokes it by default
            runPCT jenkins: fileUri, metadataFile: metadataPath,
                    pctUrl: "docker://jenkins/pct:pr74",
                    javaOptions: ["-Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn"]
            deleteDir()
        }
    }
}