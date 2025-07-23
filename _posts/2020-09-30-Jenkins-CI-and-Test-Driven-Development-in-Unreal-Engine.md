# Jenkins, CI and Test-Driven Development

September 30th, 2020.

# Requirements

* Windows 10.

* [Visual Studio](https://visualstudio.microsoft.com/vs/).

* [Unreal Engine 4](https://www.unrealengine.com/en-US/get-now) (4.22+).

* [git](https://git-scm.com/).

* [GitHub repository](https://github.com/).

* [ngrok](https://ngrok.com/).

* [Jenkins](https://www.jenkins.io/).

* [OpenCppCoverage](https://github.com/OpenCppCoverage/OpenCppCoverage).

I assume you understand how Test-Driven Development (TDD) and Continuous Integration (CI) works. If not, check the resources section, it's a nice place to start and you can try tests with this project.

I created a [project template](https://github.com/Floating-Island/UE4-TDD-CI_Testing) that you can use however you like. It' was originallly made in 4.22 but it got updated to 4.25.

# Introduction

Unreal Engine provides a testing suite inside it's Automation Framework, but it's tedious to write a test, build the project, open the editor, run the tests and see if they pass or fail.

There's a way to do the tests more efficiently (you still have to create a class from within the editor to use it as a test class so the project 'sees' it), without having to wait the editor to finish and check the results for yourself.

What you need  is Jenkins, an automation program that triggers a pipeline execution via an event. A pipeline is a configuration of a workspace environment, a series of stages, each of them consisting of a series of steps (calling batch files in windows, executing commands, printing logs, etc), and finally things that you do after (post) the pipeline is executed.

Inside that pipeline we're going to declare how to build the project, run our tests, check if they fail or pass and also which parts of the project aren't being tested (via code coverage).

How's the process then?

 1. You code locally (create tests, classes, etc).

 2. You commit code.

 3. You push your code (or do a pull request).

 4. Github receives the push and uses it's webhook to notify Jenkins via a tunnel created by ngrok (because we don't have a way to communicate directly with Jenkins).

 5. Jenkins receives a notification that a repository included in a pipeline has received a push.

 6. Jenkins pulls every change to the repository in Jenkins workspace.

 7. Jenkins starts the pipeline associated with that repository.

 8. The Pipeline builds the project.

 9. The Pipeline runs the tests while doing code coverage.

10. The Pipeline shows build status and tests reports.

11. Jenkins notifies Github the results of the pipeline build.

Looks easy, right? The only problem is understanding that Jenkins is meant to be used in a server, which means that it (and every application that the pipeline invokes) has to work in headless mode. Also, no application invoked has to have any input allowed.

This problem is a source of headaches in the beginning, but you'll become accustomed to it.

# Path:

 1. Install required programs.

 2. Create Unreal Project.

 3. Add .gitignore.

 4. Add Jenkinsfile and push changes.

 5. Create a class (without parent, None) from the UE Editor, place it in a separate 'Tests' folder and use it as a test class.

 6. Create tests.

 7. In Jenkins Install:

    * Blue Ocean plugin (there're plugins necessary with it and if you want a prettier Jenkins).

    * GitHub plugin (for pull requests).

    * HTTP request plugin (mm don't know if necessary, but it was some time ago).

    * Cobertura plugin (for code coverage).

    * Slack plugin and configure it (if you want slack notifications).

 8. Create Jenkins Multibranch Pipeline project.

 9. Create a tunnel via ngrok to the Jenkins port (default is 8080).

10. Add a webhook to the github repository referencing the http given by ngrok (don't forget to add a forward slash '/' to the webhook trail if it doesn't have one!!!).

11. Push to see the build trigger in Jenkins.

It would be nice to add github checks to pull requests, but it's only possible with a paid account in private repositories.

# Steps:

### 1)Install Jenkins

Head to the [Jenkins download page](https://www.jenkins.io/download/) and install it following the installer steps.

Open Jenkins via a new tab inside your browser (by default, Jenkins is at http://localhost:8080/ ).

Inside it, go to Manage Jenkins (on the left pane), then to Manage Plugins, then to the Available tab and search and install the following plugins:

* Blue Ocean plugin (there're plugins necessary with it and  it's a nice addition if you want a prettier Jenkins).

* GitHub plugin (for pull requests).

* HTTP request plugin (mm don't know if necessary, but it was some time ago).

* Cobertura plugin (for code coverage).

* Slack plugin and configure it (if you want slack notifications).

### 2) Github & UE4

I asume you have a UE4 project created, added a .gitignore file to it, added the project to source control and pushed the local repository to a GitHub repository.

### 3) OpenCppCoverage

Download and install it from the [releases page](https://github.com/OpenCppCoverage/OpenCppCoverage/releases) and remember its installation path.

### 4) Jenkinsfile

If you don't want to create the jenkinsfile from scratch, you can use the one inside the GitHub [repository](https://github.com/Floating-Island/UE4-TDD-CI_Testing) that works when you have done the rest of the steps on this guide. You can use that project as template and skip this step.

The jenkinsfile is the pipeline of the project. In there goes every step and configuration that you need to automate (like building, testing, reports creation, etc). It has to be inside the project folder for Jenkins to be able to execute it.

It looks something like this:

```
pipeline {
  //configuration...


//pipeline execution
  stages {//the pipeline stages

    stage('Building') {//a pipeline stage called 'Building'

      steps {//steps made in the 'Building' stage

        echo 'Build Stage Started.'//a step in the stage
        bat "buildWithoutCooking.bat"//another step in the stage
      }//end of the 'building' steps

      post {//actions made after the stage execution
        
       //things that will be done always, after the stage execution...

        success {//things that will be done only if the stage execution succeeds:
          echo 'Build Stage Successful.'
        }
        failure {//things that will be done only if the stage fails:
          echo 'Build Stage Unsuccessful.'
        }
      }
    }

    stage('Testing') {
      steps {
        echo 'Testing Stage Started.'
        bat "TestRunner.bat"
      }
      post {
        success {
          echo 'Testing Stage Successful.'
        }
        failure {
          echo 'Testing Stage Unsuccessful.'
        }
      }
    }
  //end of stages
  }
//end of pipeline
}
```

You can use Post after the end of stages too and it'll be executed after the pipeline executes (even if it fails).

We're going to build upon this file and spend the majority of time here.

**-Setting a workspace:**

```
agent {
    node {
      label 'master'
      customWorkspace "C:\\ProjectRWorkspace"//use backward slashes to avoid problems with how Windows uses directories!!
    }
  }//^all this is necessary to run the build in a special workspace.
```

It goes inside the //configuration section.

Here we're just setting a custom workspace with the declaration of `customworkspace`.

Agent is the machine wiil execute the pipeline/task. Node is the same, but they are used on different types of pipelines (agent for declarative ones and node for scripted ones). In this case, we need to combine the two to specify which machine runs the pipeline and, more importantly, where it's going to be executed.

Label is used to specify the name of the node/computer that will do the following tasks. You can see your nodes inside Jenkins if you go to Manage Jenkins and then to Manage Nodes and Clouds.

In this case I use node 'master' because it's the only one I have.

But, why do we want a custom workspace?

Well Unreal Engine doesn't like long file paths and the ones that Jenkins uses are normally long. So we specify a workspace folder closest to the drive to avoid problems.

The jenkinsfile should look like this now:

```
pipeline {
  agent {node {
        label 'master'
        customWorkspace "D:/newPlace"
    }}//^all this is necessary to run the build in a special workspace.
  stages {
    stage('Building') {
      steps {
        echo 'Build Stage Started.'
        bat "buildWithoutCooking.bat"
      }
      post {
        success {
          echo 'Build Stage Successful.'
        }
        failure {
          echo 'Build Stage Unsuccessful.'
        }
      }
    }

    stage('Testing') {
      steps {
        echo 'Testing Stage Started.'
        bat "newTestRunner.bat"
      }
      post {
        success {
          echo 'Testing Stage Successful.'
        }
        failure {
          echo 'Testing Stage Unsuccessful.'
        }
      }
    }

  }
}
```

**-Setting environment variables:**

```
  environment {
    ue4Path = "C:\\Program Files\\Epic Games\\UE_4.22"
    ue4Project = "CITesting"
    ueProjectFileName = "${ue4Project}.uproject"
    testSuiteToRun = "Game."//the '.' is used to run all tests inside the prettyname. The automation system searches for everything that has 'Game.' in it, so otherGame.'s tests would run too...
    testReportFolder = "TestsReport"
    testsLogName = "RunTests.log"
    pathToTestsLog = "${env.WORKSPACE}" + "\\Saved\\Logs\\" + "${testsLogName}"
    codeCoverageReportName="CodeCoverageReport.xml"
  }
```

This will be used later on the pipeline. With these, you avoid repeating paths and typos.

This too goes into the //configuration section.

Updated jenkinsfile:

```
pipeline {
    agent {node {
        label 'master'
        customWorkspace "D:/newPlace"
    }}//^all this is necessary to run the build in a special workspace.

    environment {
        ue4Path = "C:\\Program Files\\Epic Games\\UE_4.22"
        ue4Project = "CITesting"
        ueProjectFileName = "${ue4Project}.uproject"
        testSuiteToRun = "Game."//the '.' is used to run all tests inside the prettyname. The automation system searches for everything that has 'Game.' in it, so otherGame.'s tests would run too...
        testReportFolder = "TestsReport"
        testsLogName = "RunTests.log"
        pathToTestsLog = "${env.WORKSPACE}" + "\\Saved\\Logs\\" + "${testsLogName}"
        codeCoverageReportName="CodeCoverageReport.xml"
    }


    stages {
        stage('Building') {
            steps {
                echo 'Build Stage Started.'
                bat "buildWithoutCooking.bat"
            }
            post {
            success {
                echo 'Build Stage Successful.'
            }
            failure {
                echo 'Build Stage Unsuccessful.'
            }
            }
        }

        stage('Testing') {
            steps {
                echo 'Testing Stage Started.'
                bat "newTestRunner.bat"
            }
            post {
            success {
                echo 'Testing Stage Successful.'
            }
            failure {
                echo 'Testing Stage Unsuccessful.'
            }
            }
        }

  }
}
```

**-Building Stage:**

Now we are going to build our project.

```
stage('Building') {
            steps {
                echo 'Build Stage Started.'
                bat "BuildWithoutCooking.bat \"${ue4Path}\" \"${env.WORKSPACE}\" \"${ueProjectFilename}\""//builds our project
            }
            post {
            success {
                echo 'Build Stage Successful.'
            }
            failure {
                echo 'Build Stage Unsuccessful.'
            }
            }
        }
```

What's important here is `bat "BuildWithoutCooking.bat \"${ue4Path}\" \"${env.WORKSPACE}\" \"${ueProjectFilename}\""`  it calls a batch file inside our project folder to build our project.

It's just a file with this inside:

```
set ue4Location=%~1
set workspace=%~2
set projectFilename=%~3


"%ue4Location%\Engine\Build\BatchFiles\RunUAT.bat" BuildCookRun -project="%workspace%\%projectFilename%" -noP4 -platform=Win64 -clientconfig=Development -build
```

it uses the arguments passed to build our project using the Unreal Automation Tool.

We use `-noP4` to tell UAT that we don't have a Perforce project.

We specify `-build` to say that we only want to build our project.

You have to open whichever text editor you like, paste the code above, save it as *BuildWithoutCooking.bat* and put it inside the project folder.

The jenkinsfile now looks like this:

```
pipeline {
    agent {node {
        label 'master'
        customWorkspace "D:/newPlace"
    }}//^all this is necessary to run the build in a special workspace.

    environment {
        ue4Path = "C:\\Program Files\\Epic Games\\UE_4.22"
        ue4Project = "CITesting"
        ueProjectFileName = "${ue4Project}.uproject"
        testSuiteToRun = "Game."//the '.' is used to run all tests inside the prettyname. The automation system searches for everything that has 'Game.' in it, so otherGame.'s tests would run too...
        testReportFolder = "TestsReport"
        testsLogName = "RunTests.log"
        pathToTestsLog = "${env.WORKSPACE}" + "\\Saved\\Logs\\" + "${testsLogName}"
        codeCoverageReportName="CodeCoverageReport.xml"
    }


    stages {
        stage('Building') {
            steps {
                echo 'Build Stage Started.'
                bat "BuildWithoutCooking.bat \"${ue4Path}\" \"${env.WORKSPACE}\" \"${ueProjectFilename}\""//builds our project
            }
            post {
                success {
                    echo 'Build Stage Successful.'
                }
                failure {
                    echo 'Build Stage Unsuccessful.'
                }
            }
        }

        stage('Testing') {
            steps {
                echo 'Testing Stage Started.'
                bat "newTestRunner.bat"
            }
            post {
            success {
                echo 'Testing Stage Successful.'
            }
            failure {
                echo 'Testing Stage Unsuccessful.'
            }
            }
        }

    }
}
```

**-Testing Stage:**

Now, two things will be happening at the same time:

* We will invoke the editor to run the tests.

* OpenCppCoverage will attach to the editor to check which files are accessed and how much code of the project files is covered while running the tests.

Our testing Stage looks as follows:

```
stage('Testing') {
            steps {
                echo 'Testing Stage Started.'

                bat "TestRunnerAndCodeCoverage.bat \"${ue4Path}\" \"${env.WORKSPACE}\" \"${ueProjectFilename}\" \"${testSuiteToRun}\" \"${testReportFolder}\" \"${testsLogName}\" \"${codeCoverageReportName}\""//runs the tests
            }
            post {
                success {
                    echo 'Testing Stage Successful.'
                }
                failure {
                    echo 'Testing Stage Unsuccessful.'
                }
            }
        }
```

And the batch file is composed of this:

```
set ue4Location=%~1
set workspace=%~2
set projectFilename=%~3
set testSuiteToRun=%~4
set testReportFolder=%~5
set testsLogName=%~6
set codeCoverageFile=%~7

set testRunnerCommand="%ue4Location%\Engine\Binaries\Win64\UE4Editor-cmd.exe" "%workspace%\%projectFilename%" -nosplash -Unattended -nopause -nosound -NullRHI -nocontentbrowser -ExecCmds="Automation RunTests %testSuiteToRun%;quit" -TestExit="Automation Test Queue Empty" -ReportOutputPath="%workspace%\%testReportFolder%" -log -Log=%testsLogName%

"C:\Program Files\OpenCppCoverage\opencppcoverage.exe" --sources=\Source --modules %workspace% --excluded_sources=\Tests --export_type=cobertura:%codeCoverageFile%  -- %testRunnerCommand%
```

*! Be sure to check that the path to OpenCppCoverage is the same as yours.*

We call the UE4Editor with this parameters:

`-nosplash` is used to not show the loading screen for UE4 editor.

`-unattended` disables feedback from the user (Jenkins runs automatically, so programs don't have to require user feedback).

`-nopause` closes the log window automatically on exit.

`-nosound` is used to run the editor without sound.

`-nullrhi` is used to tell the editor that we don't want to render anything, like a headless mode.

`-nocontentbrowser` does that, it tells to disable the content browser.

`-execcmds` executes the quoted commands. In this case tells the Automation module to run the tests that match the word stored in our testsuite variable. It then quits the editor.

`-testexit` is used to tell Automation when to stop. In this case it will stop when it finds that the test queue is empty.

`-reportoutputpath` tells where to store the test report.

`-log` opens a separate window to show the log contents.

`-log=` tells where to store the log.

We call OpenCppCoverage with this parameters:

`--sources` tells where are the source files stored.

`--modules` does the same job as sources, but for the executable and the shared libraries.

`--export_type` tells what report format we want to generate. In this case we'll use the Cobertura format to be able to parse it into Jenkins.

The last separated `--` is used to tell OpenCppCoverage that we have finished declaring parameters and the next thing specified is the program that executes the tests (UE4Editor in this case).

Just save the file as TestRunnerAndCodeCoverage.bat and put it inside the project folder.

The jenkinsfile should be like:

```
pipeline {
    agent {node {
        label 'master'
        customWorkspace "D:/newPlace"
    }}//^all this is necessary to run the build in a special workspace.

    environment {
        ue4Path = "C:\\Program Files\\Epic Games\\UE_4.22"
        ue4Project = "CITesting"
        ueProjectFileName = "${ue4Project}.uproject"
        testSuiteToRun = "Game."//the '.' is used to run all tests inside the prettyname. The automation system searches for everything that has 'Game.' in it, so otherGame.'s tests would run too...
        testReportFolder = "TestsReport"
        testsLogName = "RunTests.log"
        pathToTestsLog = "${env.WORKSPACE}" + "\\Saved\\Logs\\" + "${testsLogName}"
        codeCoverageReportName="CodeCoverageReport.xml"
    }


    stages {
        stage('Building') {
            steps {
                echo 'Build Stage Started.'
                bat "BuildWithoutCooking.bat \"${ue4Path}\" \"${env.WORKSPACE}\" \"${ueProjectFilename}\""//builds our project
            }
            post {
                success {
                    echo 'Build Stage Successful.'
                }
                failure {
                    echo 'Build Stage Unsuccessful.'
                }
            }
        }

        stage('Testing') {
            steps {
                echo 'Testing Stage Started.'

                bat "TestRunnerAndCodeCoverage.bat \"${ue4Path}\" \"${env.WORKSPACE}\" \"${ueProjectFilename}\" \"${testSuiteToRun}\" \"${testReportFolder}\" \"${testsLogName}\" \"${codeCoverageReportName}\""//runs the tests
            }
            post {
                success {
                    echo 'Testing Stage Successful.'
                }
                failure {
                    echo 'Testing Stage Unsuccessful.'
                }
            }
        }
    }
}
```

**-Generating JUnit Tests and Code Coverage Report:**

After all the stages are executed, we'll generate the reports and publish them to Jenkins.

This goes in the post section:

```
post {
    always{
      echo 'Tests finished, printing log.'
      bat "type ${pathToTestsLog}"
      echo 'Formatting TestsReport from JSon to JUnit XML'
      formatUnitTests()

      echo "Publish Code Coverage Report."
      cobertura(coberturaReportFile:"${codeCoverageReportName}")
  }
```

And outside the pipeline you put this code to format the tests that it's a slightly modified version from the one found in Michael Delva's [blog](https://www.emidee.net/ue4/2018/11/13/UE4-Unit-Tests-in-Jenkins.html):

```
import groovy.json.JsonSlurper
import groovy.xml.MarkupBuilder

def testReportSummary = 'to be populated...'

def formatUnitTests() {
        convertTestsReport()
        testReportSummary = junit "${testReportFolder}\\junit.xml"
}

def convertTestsReport() {
    def jsonReport = readFile file: "${testReportFolder}\\index.json", encoding: "UTF-8"
    // Needed because the JSON is encoded in UTF-8 with BOM

    jsonReport = jsonReport.replace( "\uFEFF", "" );

    def xmlContent = transformReport( jsonReport )

    writeFile file: "${testReportFolder}\\junit.xml", text: xmlContent.toString()
}

@NonCPS//atomic method
def transformReport( String jsonContent ) {

    def parsedReport = new JsonSlurper().parseText( jsonContent )
    
    def jUnitReport = new StringWriter()
    def builder = new MarkupBuilder( jUnitReport )

    builder.doubleQuotes = true
    builder.mkp.xmlDeclaration version: "1.0", encoding: "utf-8"

    builder.testsuite( tests: parsedReport.succeeded + parsedReport.failed, failures: parsedReport.failed, time: parsedReport.totalDuration ) {
      for ( test in parsedReport.tests ) {
        builder.testcase( name: test.testDisplayName, classname: test.fullTestPath, status: test.state ) {
          if(test.state == "Fail") {
            for ( entry in test.entries ) { 
              if(entry.event.type == "Error") {
                builder.failure( message: entry.event.message, type: entry.event.type, entry.filename + " " + entry.lineNumber )
              }
            }
          }
        }
      }
    } 

    return jUnitReport.toString()
}
```

And your Jenkinsfile should look like something like this:

```
pipeline {
    agent {node {
        label 'master'
        customWorkspace "D:/newPlace"
    }}//^all this is necessary to run the build in a special workspace.

    environment {
        ue4Path = "C:\\Program Files\\Epic Games\\UE_4.22"
        ue4Project = "CITesting"
        ueProjectFileName = "${ue4Project}.uproject"
        testSuiteToRun = "Game."//the '.' is used to run all tests inside the prettyname. The automation system searches for everything that has 'Game.' in it, so otherGame.'s tests would run too...
        testReportFolder = "TestsReport"
        testsLogName = "RunTests.log"
        pathToTestsLog = "${env.WORKSPACE}" + "\\Saved\\Logs\\" + "${testsLogName}"
        codeCoverageReportName="CodeCoverageReport.xml"
    }


    stages {
        stage('Building') {
            steps {
                echo 'Build Stage Started.'
                bat "BuildWithoutCooking.bat \"${ue4Path}\" \"${env.WORKSPACE}\" \"${ueProjectFilename}\""//builds our project
            }
            post {
                success {
                    echo 'Build Stage Successful.'
                }
                failure {
                    echo 'Build Stage Unsuccessful.'
                }
            }
        }

        stage('Testing') {
            steps {
                echo 'Testing Stage Started.'

                bat "TestRunnerAndCodeCoverage.bat \"${ue4Path}\" \"${env.WORKSPACE}\" \"${ueProjectFilename}\" \"${testSuiteToRun}\" \"${testReportFolder}\" \"${testsLogName}\" \"${codeCoverageReportName}\""//runs the tests
            }
            post {
                success {
                    echo 'Testing Stage Successful.'
                }
                failure {
                    echo 'Testing Stage Unsuccessful.'
                }
            }
        }
    }


    post {
    always{
      echo 'Tests finished, printing log.'
      bat "type ${pathToTestsLog}"
      echo 'Formatting TestsReport from JSon to JUnit XML'
      formatUnitTests()

      echo "Publish Code Coverage Report."
      cobertura(coberturaReportFile:"${codeCoverageReportName}")
  }


}


import groovy.json.JsonSlurper
import groovy.xml.MarkupBuilder

def testReportSummary = 'to be populated...'

def formatUnitTests() {
        convertTestsReport()
        testReportSummary = junit "${testReportFolder}\\junit.xml"
}

def convertTestsReport() {
    def jsonReport = readFile file: "${testReportFolder}\\index.json", encoding: "UTF-8"
    // Needed because the JSON is encoded in UTF-8 with BOM

    jsonReport = jsonReport.replace( "\uFEFF", "" );

    def xmlContent = transformReport( jsonReport )

    writeFile file: "${testReportFolder}\\junit.xml", text: xmlContent.toString()
}

@NonCPS//atomic method
def transformReport( String jsonContent ) {

    def parsedReport = new JsonSlurper().parseText( jsonContent )
    
    def jUnitReport = new StringWriter()
    def builder = new MarkupBuilder( jUnitReport )

    builder.doubleQuotes = true
    builder.mkp.xmlDeclaration version: "1.0", encoding: "utf-8"

    builder.testsuite( tests: parsedReport.succeeded + parsedReport.failed, failures: parsedReport.failed, time: parsedReport.totalDuration ) {
      for ( test in parsedReport.tests ) {
        builder.testcase( name: test.testDisplayName, classname: test.fullTestPath, status: test.state ) {
          if(test.state == "Fail") {
            for ( entry in test.entries ) { 
              if(entry.event.type == "Error") {
                builder.failure( message: entry.event.message, type: entry.event.type, entry.filename + " " + entry.lineNumber )
              }
            }
          }
        }
      }
    } 

    return jUnitReport.toString()
}
```

**-Workspace cleanup:**

Now that all the work has been done, we should clean the workspace, so the next build works in a clean environment.

Here, we'll keep only the repository files that Jenkins initially downloaded, while destroying everything else.

You should take into consideration how are your pipeline times, because it could be possible that in the future you would need your build files to speed up build times, instead of making a clean build each time.

So, the code for cleanup is as follows and should be put inside the post section, but after we publish the code coverage reports:

```
      post {
    always{
      echo 'Tests finished, printing log.'
      bat "type ${pathToTestsLog}"
      echo 'Formatting TestsReport from JSon to JUnit XML'
      formatUnitTests()

      echo "Publish Code Coverage Report."
      cobertura(coberturaReportFile:"${codeCoverageReportName}")


      echo 'Cleaning up workspace:'
      echo '-checking current workspace.'
      powershell label: 'show workspace', script: 'dir $WORKSPACE'
      bat 'git reset --hard'//resets to HEAD, to the commit in the cloned repository.
      bat 'git clean -dffx .'//removes untracked files.
      echo '-checking clean workspace.'
      powershell label: 'show workspace', script: 'dir $WORKSPACE'
  }
```

What this does is call git to reset to HEAD, the commit we received when downloading the changes. It then removes the untracked files (everything we did after downloading the changes).

The jenkinsfile should look like this:

```
pipeline {
    agent {node {
        label 'master'
        customWorkspace "D:/newPlace"
    }}//^all this is necessary to run the build in a special workspace.

    environment {
        ue4Path = "C:\\Program Files\\Epic Games\\UE_4.22"
        ue4Project = "CITesting"
        ueProjectFileName = "${ue4Project}.uproject"
        testSuiteToRun = "Game."//the '.' is used to run all tests inside the prettyname. The automation system searches for everything that has 'Game.' in it, so otherGame.'s tests would run too...
        testReportFolder = "TestsReport"
        testsLogName = "RunTests.log"
        pathToTestsLog = "${env.WORKSPACE}" + "\\Saved\\Logs\\" + "${testsLogName}"
        codeCoverageReportName="CodeCoverageReport.xml"
    }


    stages {
        stage('Building') {
            steps {
                echo 'Build Stage Started.'
                bat "BuildWithoutCooking.bat \"${ue4Path}\" \"${env.WORKSPACE}\" \"${ueProjectFilename}\""//builds our project
            }
            post {
                success {
                    echo 'Build Stage Successful.'
                }
                failure {
                    echo 'Build Stage Unsuccessful.'
                }
            }
        }

        stage('Testing') {
            steps {
                echo 'Testing Stage Started.'

                bat "TestRunnerAndCodeCoverage.bat \"${ue4Path}\" \"${env.WORKSPACE}\" \"${ueProjectFilename}\" \"${testSuiteToRun}\" \"${testReportFolder}\" \"${testsLogName}\" \"${codeCoverageReportName}\""//runs the tests
            }
            post {
                success {
                    echo 'Testing Stage Successful.'
                }
                failure {
                    echo 'Testing Stage Unsuccessful.'
                }
            }
        }
    }


    post {
    always{
      echo 'Tests finished, printing log.'
      bat "type ${pathToTestsLog}"
      echo 'Formatting TestsReport from JSon to JUnit XML'
      formatUnitTests()

      echo "Publish Code Coverage Report."
      cobertura(coberturaReportFile:"${codeCoverageReportName}")


      echo 'Cleaning up workspace:'
      echo '-checking current workspace.'
      powershell label: 'show workspace', script: 'dir $WORKSPACE'
      bat 'git reset --hard'//resets to HEAD, to the commit in the cloned repository.
      bat 'git clean -dffx .'//removes untracked files.
      echo '-checking clean workspace.'
      powershell label: 'show workspace', script: 'dir $WORKSPACE'
  }


}


import groovy.json.JsonSlurper
import groovy.xml.MarkupBuilder

def testReportSummary = 'to be populated...'

def formatUnitTests() {
        convertTestsReport()
        testReportSummary = junit "${testReportFolder}\\junit.xml"
}

def convertTestsReport() {
    def jsonReport = readFile file: "${testReportFolder}\\index.json", encoding: "UTF-8"
    // Needed because the JSON is encoded in UTF-8 with BOM

    jsonReport = jsonReport.replace( "\uFEFF", "" );

    def xmlContent = transformReport( jsonReport )

    writeFile file: "${testReportFolder}\\junit.xml", text: xmlContent.toString()
}

@NonCPS//atomic method
def transformReport( String jsonContent ) {

    def parsedReport = new JsonSlurper().parseText( jsonContent )
    
    def jUnitReport = new StringWriter()
    def builder = new MarkupBuilder( jUnitReport )

    builder.doubleQuotes = true
    builder.mkp.xmlDeclaration version: "1.0", encoding: "utf-8"

    builder.testsuite( tests: parsedReport.succeeded + parsedReport.failed, failures: parsedReport.failed, time: parsedReport.totalDuration ) {
      for ( test in parsedReport.tests ) {
        builder.testcase( name: test.testDisplayName, classname: test.fullTestPath, status: test.state ) {
          if(test.state == "Fail") {
            for ( entry in test.entries ) { 
              if(entry.event.type == "Error") {
                builder.failure( message: entry.event.message, type: entry.event.type, entry.filename + " " + entry.lineNumber )
              }
            }
          }
        }
      }
    } 

    return jUnitReport.toString()
}
```

Now, we are ready to create a Multi-Branch Pipeline in Jenkins.

### 5) Creating a Jenkins Multi-Branch Pipeline

Back into Jenkins, we go to the left pane and Open Blue Ocean.

In the right, we create a new pipeline and follow the steps.

When it prompts to use github, you should create credentials for your account and authorize it from GitHub.

Now, Jenkins is able to build from GitHub. But GitHub doesn't know where to send pushes or pull requests so Jenkins will be triggered by them. That's why we need to use GitHub webhooks inside our GitHub Project.

**!!! You'll need to manually approve some scripts so Jenkins is able to parse the reports !!!**

Here's how it's done:

* Inside Jenkins, go to Dashboard, then to Manage Jenkins.

* In there, below Security, go to In-process Script Approval.

* Add these scripts:

  * method groovy.lang.GroovyObject invokeMethod java.lang.String java.lang.Object

  * method groovy.xml.MarkupBuilder getMkp

  * method groovy.xml.MarkupBuilder setDoubleQuotes boolean

  * method groovy.xml.MarkupBuilderHelper xmlDeclaration java.util.Map

  * new groovy.xml.MarkupBuilder java.io.Writer

* Click on Approve and done!

If you don't, the build will fail and you will be notified only if you go to Dashboard -> Manage Jenkins ->  In-process Script Approval, each time a script needs to be approved.

Thanks Scott (sradms0) for the heads up!

### 6)GitHub Webhooks to trigger Jenkins builds:

Go to your GitHub repository, then settings, then Webhooks and add a new one.

In the Webhook configuration change the 'Which events would you like to trigger this webhook?' and select 'Let me select individual events.', select push events and pull requests.

Now, we need a URL to notify of this events, but Jenkins works locally, in our computer.

What we need to do is use ngrok to open a tunnel from a public IP to the port Jenkins listens to.

### 7) ngrok

Download and install ngrok from the [download page](https://ngrok.com/download).

Run ngrok and create a tunnel (open a port) to use with Jenkins. It has to be the same port that you use when accessing Jenkins (port 8080 by default).

The command to create a tunnel is as follows:

```
ngrok http portNumber
```

In our case (as Jenkins default) is as follows:

```
ngrok http 8080
```

Ngrok will create a HTTP URL just for us and show it on the next window. That URL is where GitHub should send it's webhooks, so it's then passed to the ngrok client which will forward it to the port where Jenkins listens.

Don't close the console because if you do, the HTTP URL will be destroyed.

### 8) Back to the GitHub Webhook

Back to the webhook we created a moment ago, we're going to paste the URL that ngrok gave us into the webhook's payload URL.

If ngrok gave something like this:

```
        http://1df3e3683768.ngrok.io
```

We'll paste it in the payload URL like this:

```
http://1df3e3683588.ngrok.io/github-webhook/
```

*Don't forget the '/' at the end.*

Save the webhook and in a moment it should appear with a tick, noting that communication is successful.

You can corroborate this heading to the ngrok console and checking that the response of Jenkins is 200 (OK).

Now Jenkins is able to receive trigger events from GitHub when someone does a Pull Request or a Push commit.

### 9) OPTIONAL - Send Slack Notifications

This is useful if you don't want to be looking at the Jenkins build progress each time it gets triggered and instead, be notified of what it's doing.

Venessa Yeh does a nice [article](https://levelup.gitconnected.com/send-slack-notifications-with-jenkins-f8e8b2d2e748) on how to setup Slack Notifications. Use it to configure Slack.

I leave the jenkinsfile of the [template repository](https://github.com/Floating-Island/UE4-TDD-CI_Testing) here so you can see a working pipeline with slack notifications:

```
pipeline {
  agent {
    node {
      label 'master'
      customWorkspace "D:\\CITestingWorkspace"//use backward slashes to avoid problems with how Windows uses directories!!
    }
  }//^all this is necessary to run the build in a special workspace.
  environment {
    ue4Path = "C:\\Program Files\\Epic Games\\UE_4.22"
    ue4Project = "CITesting"
    ueProjectFileName = "${ue4Project}.uproject"
    testSuiteToRun = "Game."//the '.' is used to run all tests inside the prettyname. The automation system searches for everything that has 'Game.' in it, so otherGame.'s tests would run too...
    testReportFolder = "TestsReport"
    testsLogName = "RunTests.log"
    pathToTestsLog = "${env.WORKSPACE}" + "\\Saved\\Logs\\" + "${testsLogName}"
    codeCoverageReportName="CodeCoverageReport.xml"
  }
  stages {
    stage('Building') {
      steps {
        echo 'Build Stage Started.'
        echo 'sending notification to Slack.'
        slackSend channel: '#testing-ci', 
          color: '#4A90E2',
          message: "Build ${env.BUILD_NUMBER} has started at node ${env.NODE_NAME}..."

        bat "BuildWithoutCooking.bat \"${ue4Path}\" \"${env.WORKSPACE}\" \"${ueProjectFilename}\""//builds our project
      }
      post {
        success {
          echo 'Build Stage Successful.'
        }
        failure {
          echo 'Build Stage Unsuccessful.'
        }
      }
    }

    stage('Testing') {
      steps {
        echo 'Testing Stage Started.'

        bat "TestRunnerAndCodeCoverage.bat \"${ue4Path}\" \"${env.WORKSPACE}\" \"${ueProjectFilename}\" \"${testSuiteToRun}\" \"${testReportFolder}\" \"${testsLogName}\" \"${codeCoverageReportName}\""//runs the tests
      }
      post {
        success {
          echo 'Testing Stage Successful.'
        }
        failure {
          echo 'Testing Stage Unsuccessful.'
        }
      }
    }


  }
  post {
    always{
      echo 'Tests finished, printing log.'
      bat "type ${pathToTestsLog}"
      echo 'Formatting TestsReport from JSon to JUnit XML'
      formatUnitTests()

        slackSend channel: "#testing-ci",
          color: '#c2f2d0',
          message: "\n *Tests Report Summary* - Total Tests: ${testReportSummary.totalCount}, Failures: ${testReportSummary.failCount}, Skipped: ${testReportSummary.skipCount}, Passed: ${testReportSummary.passCount}"

      echo "Publish Code Coverage Report."
      cobertura(coberturaReportFile:"${codeCoverageReportName}")

      echo 'Cleaning up workspace:'
      echo '-checking current workspace.'
      powershell label: 'show workspace', script: 'dir $WORKSPACE'
      bat 'git reset --hard'//resets to HEAD, to the commit in the cloned repository.
      bat 'git clean -dffx .'//removes untracked files.
      echo '-checking clean workspace.'
      powershell label: 'show workspace', script: 'dir $WORKSPACE'

      echo 'Sending build status notification to Slack:'
    }
    success{
        slackSend channel: '#testing-ci',
          color: 'good', 
          message: "*${currentBuild.currentResult}:* Build ${env.BUILD_NUMBER} has *succeded!* :innocent:"
    }
    unstable{
        slackSend channel: '#testing-ci',
          color: '#E2A52E', 
          message: "*${currentBuild.currentResult}:* Build ${env.BUILD_NUMBER} it's *unstable!* :grimacing:"
    }
    failure{
        slackSend channel: '#testing-ci',
          color: 'danger', 
          message: "*${currentBuild.currentResult}:* Build ${env.BUILD_NUMBER} has *failed* :astonished:"
    }
  }
}

import groovy.json.JsonSlurper
import groovy.xml.MarkupBuilder

def testReportSummary = 'to be populated...'

def formatUnitTests() {
        convertTestsReport()
        testReportSummary = junit "${testReportFolder}\\junit.xml"
}

def convertTestsReport() {
    def jsonReport = readFile file: "${testReportFolder}\\index.json", encoding: "UTF-8"
    // Needed because the JSON is encoded in UTF-8 with BOM

    jsonReport = jsonReport.replace( "\uFEFF", "" );

    def xmlContent = transformReport( jsonReport )

    writeFile file: "${testReportFolder}\\junit.xml", text: xmlContent.toString()
}

@NonCPS//atomic method
def transformReport( String jsonContent ) {

    def parsedReport = new JsonSlurper().parseText( jsonContent )
    
    def jUnitReport = new StringWriter()
    def builder = new MarkupBuilder( jUnitReport )

    builder.doubleQuotes = true
    builder.mkp.xmlDeclaration version: "1.0", encoding: "utf-8"

    builder.testsuite( tests: parsedReport.succeeded + parsedReport.failed, failures: parsedReport.failed, time: parsedReport.totalDuration ) {
      for ( test in parsedReport.tests ) {
        builder.testcase( name: test.testDisplayName, classname: test.fullTestPath, status: test.state ) {
          if(test.state == "Fail") {
            for ( entry in test.entries ) { 
              if(entry.event.type == "Error") {
                builder.failure( message: entry.event.message, type: entry.event.type, entry.filename + " " + entry.lineNumber )
              }
            }
          }
        }
      }
    } 

    return jUnitReport.toString()
}
```

### 10) Want some useful tests??

Since before creating this tutorial I've been working on my final project for university.

The project was about a simple racing videogame using TDD and CI as the environment.

I made it public so others will be able to have even more examples of tests implementations with TDD (over 340!!).

It also has a slightly modified version of this tutorial jenkinsfile, replication tests over LAN and some helpful classes to aid on the tests.

Also, it comes with the logical view and a report (currently in Spanish but I'll upload the English version soon).

[Project Repository](https://github.com/Floating-Island/ProjectR)

I hope this really helps in your projects! Enjoy!!

### 11) The End

If you need to create C++ tests and you don't know where to start, you can read my articles on [Considerations On Testing UE4 Classes](https://unrealcommunity.wiki/considerations-on-testing-ue4-classes-y45ygvk0).

Bye!

Alberto Mikulan

# Resources

The following is a list of resources that helped me while doing this article/project.

# TDD

* [Test-Driven Development - Wikipedia](https://en.wikipedia.org/wiki/Test-driven_development)

* [Self Testing Code - Martin Fowler](https://martinfowler.com/bliki/SelfTestingCode.html)

* [Test-Driven Development - Martin Fowler](https://martinfowler.com/bliki/TestDrivenDevelopment.html)

# CI

* [Continuous Integration - Wikipedia](https://en.wikipedia.org/wiki/Continuous_integration)

* [Continuous Integration - Martin Fowler](https://martinfowler.com/articles/continuousIntegration.html)

* [Continuous Integration - Thoughtworks](https://www.thoughtworks.com/continuous-integration)

* [Continuous Delivery - Martin Fowler](https://martinfowler.com/bliki/ContinuousDelivery.html?utm_source=Codeship&utm_medium=CI-Guide)

* [Continuous Integration vs. Continuous Delivery vs. Continuous Deployment - Atlassian](https://www.atlassian.com/continuous-delivery/principles/continuous-integration-vs-delivery-vs-deployment)

* [Extra: Branching Patterns - Martin Fowler](https://martinfowler.com/articles/branching-patterns.html)

# Jenkins

* [How force Jenkins to show UI always in English? Issue - Superuser - sobi3ch](https://superuser.com/questions/879392/how-force-jenkins-to-show-ui-always-in-english)

* [Jenkins Automated Build Trigger On Github Pull Request - DevOpsCube](https://devopscube.com/jenkins-build-trigger-github-pull-request/)

* [Jenkins Multibranch Pipeline Tutorial For Beginners - DevOpsCube](https://devopscube.com/jenkins-multibranch-pipeline-tutorial/)

* [Pipeline - Jenkins](https://www.jenkins.io/doc/book/pipeline/)

* [Pipeline Syntax - Jenkins](https://www.jenkins.io/doc/book/pipeline/syntax/)

* [Pipeline Syntax: Agent - Jenkins](https://www.jenkins.io/doc/book/pipeline/syntax/#agent)

* [Pipeline Examples - Jenkins](https://www.jenkins.io/doc/pipeline/examples/)

* [Using a Jenkinsfile - Jenkins](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/)

* [Jenkins : how to print the contents of a text file to the build log? Issue - StackOverflow - AnneTheAgile](https://stackoverflow.com/questions/27453156/jenkins-how-to-print-the-contents-of-a-text-file-to-the-build-log)

* [jenkins pipeline: agent vs node? Issue - StackOverflow - Matthias M](https://stackoverflow.com/questions/42050626/jenkins-pipeline-agent-vs-node)

* [How do I make Jenkins pipeline run in (any) agent machine, but never master? Issue - StackOverflow - Joshua Fox](https://stackoverflow.com/questions/54020776/how-do-i-make-jenkins-pipeline-run-in-any-agent-machine-but-never-master)

* [Why our batch process works locally and not from Jenkins? - CloudBees](https://support.cloudbees.com/hc/en-us/articles/115000733051-Why-our-batch-process-works-locally-and-not-from-Jenkins-)

* [Jenkins CI Pipeline Scripts not permitted to use method groovy.lang.GroovyObject Issue - StackOverflow - Daniel Hernández](https://stackoverflow.com/questions/38276341/jenkins-ci-pipeline-scripts-not-permitted-to-use-method-groovy-lang-groovyobject)

* [Processing XML - Apache Groovy](https://groovy-lang.org/processing-xml.html)

* [Jenkins Workflow-Plugin and Groovy Libs Issue - StackOverflow - Rene](https://stackoverflow.com/questions/33912964/jenkins-workflow-plugin-and-groovy-libs)

# Slack notifications

* [Send Slack Notifications with Jenkins - Venessa Yeh](https://levelup.gitconnected.com/send-slack-notifications-with-jenkins-f8e8b2d2e748)

* [Use emoji and emoticons - Slack](https://slack.com/intl/en-ar/help/articles/202931348-Use-emoji-and-emoticons)

* [Emoji Cheat Sheet - Webfx](https://www.webfx.com/tools/emoji-cheat-sheet/)

* [Slack Notification Plugin - Jenkins](https://www.jenkins.io/doc/pipeline/steps/slack/)

* [Color Hex Color Codes - color-hex](https://www.color-hex.com/)

# ngrok

* [ngrok Official page](https://ngrok.com/product)

* [An ngrok Tutorial and Primer - Daniel Miessler](https://danielmiessler.com/study/ngrok/)

* [Secure localhost tunnels with ngrok - Atlassian](https://blog.developer.atlassian.com/secure-localhost-tunnels-with-ngrok/)

# GitHub

* [Creating Webhooks - GitHub](https://developer.github.com/webhooks/creating/)

* [Triggering a Jenkins build on push using GitHub webhooks - Parul Dixit](https://medium.com/faun/triggering-jenkins-build-on-push-using-github-webhooks-52d4361542d4)

* [Github webhook with local Jenkins and ngrok Issue - StackOverflow - Pruitlgoe](https://stackoverflow.com/questions/58402423/github-webhook-with-local-jenkins-and-ngrok)

* [Github Webhook With Jenkins return 302 NotFound Issue - StackOverflow - Xiaoxi Bian](https://stackoverflow.com/questions/49848884/github-webhook-with-jenkins-return-302-notfound)

# OpenCppCoverage

* [OpenCppCoverage](https://github.com/OpenCppCoverage/OpenCppCoverage)

* [OpenCppCoverage and Cobertura Plugin](https://github.com/OpenCppCoverage/OpenCppCoverage/wiki/Jenkins)

# UE4

* [Automation System Overview - Epic Games](https://docs.unrealengine.com/en-US/Programming/Automation/index.html)

* [Automation System User Guide](https://docs.unrealengine.com/en-US/Programming/Automation/UserGuide/index.html)

* [Automation Technical Guide - Epic Games](https://docs.unrealengine.com/en-US/Programming/Automation/TechnicalGuide/index.html)

* [How Rare Tests Sea of Thieves to Stop Bugs Reaching Players | AI and Games - YouTube](https://www.youtube.com/watch?v=Bu4YV4be6IE)

* [Automated Testing at Scale in Sea of Thieves | Unreal Fest Europe 2019 | Unreal Engine - YouTube - Jessica Baker](https://www.youtube.com/watch?v=KmaGxprTUfI)

* [Tech Blog: Tests and Testability - Rare - Jessica Baker](https://www.rare.co.uk/news/tech-blog-testability)

* [Continuous Delivery in Games - Jafar Soltani](https://www.gdcvault.com/play/1025318/Adopting-Continuous)

* [Tech Blog: Adopting Continuous Delivery (Part 1) - Rare - Jafar Soltani](https://www.rare.co.uk/news/tech-blog-continuous-delivery-part1)

* [Tech Blog: Adopting Continuous Delivery (Part 2) - Rare - Jafar Soltani](https://www.rare.co.uk/news/tech-blog-continuous-delivery-part2)

* [Tech Blog: Adopting Continuous Delivery (Part 3) - Rare - Jafar Soltani](https://www.rare.co.uk/news/tech-blog-continuous-delivery-part3)

* [Automated Testing of Gameplay Features in 'Sea of Thieves' - Robert Masella](https://www.gdcvault.com/play/1026366/Automated-Testing-of-Gameplay-Features)

* [Automation with Unreal Engine and Jenkins-CI - Patrice Vignola](https://patricevignola.com/post/automation-jenkins-unreal)

* [Unreal Build Automation and Deployment at ExtroForge - ExtroForge](http://www.extroforge.com/unreal-build-automation-and-deployment-at-extroforge/)

* [Jenkins CI Automation for Unreal Engine 4 Projects - GitHub - Skymap Games](https://github.com/skymapgames/jenkins-ue4)

* [UE4 Unit Tests in Jenkins - Michael Delva](https://www.emidee.net/ue4/2018/11/13/UE4-Unit-Tests-in-Jenkins.html)

* [UE4 Automation Tool - Jonathan Hale](https://blog.squareys.de/ue4-automation-tool/)

* [Who has the latest Unreal Engine 4 Console Variables and Commands? Can you share it with me? Issue - Unreal Engine Forums - GaoHeShun](https://answers.unrealengine.com/questions/751432/view.html)

* [Command-Line Arguments - Epic Games](https://docs.unrealengine.com/en-US/Programming/Basics/CommandLineArguments/index.html)

* [Console Variables in C++ - Epic Games](https://docs.unrealengine.com/en-US/Programming/Development/Tools/ConsoleManager/index.html)

* [Automate deployment with the Unreal Engine using the Unreal Automation Tool (UAT) - Marvin Pohl](https://blog.mi.hdm-stuttgart.de/index.php/2017/02/11/uat-automation/)

* [Unit Tests in Unreal – pt 1 – A definition of Unit Tests for this series - Eric Lemes](https://ericlemes.com/2018/12/12/unit-tests-in-unreal-pt-1/)

* [Simple Automation Test on Dedicated Server Build Issue - rjjatson](https://answers.unrealengine.com/questions/706252/simple-automation-test-on-dedicated-server-build.html)

* [Run automated testing from command line Discussion - nikitablack](https://answers.unrealengine.com/questions/106978/run-automated-testing-from-command-line.html)

* [Unreal Engine 4. Different ways to instantiate the object Discussion - StackOverflow - raviga](https://stackoverflow.com/questions/60021804/unreal-engine-4-different-ways-to-instantiate-the-object)

* [Unreal Containers (future use)](https://unrealcontainers.com/docs/use-cases/continuous-integration)

# BATCH

* [Pass Command Line arguments (Parameters) to a Windows batch file - ss64](https://ss64.com/nt/syntax-args.html)

* [Remove Quotes from a string - ss64](https://ss64.com/nt/syntax-dequote.html)

* [Setting variables - ss64](https://ss64.com/nt/set.html)

* [Defining and using a variable in batch file - StackOverflow - Jamie Dixon](https://stackoverflow.com/questions/10552812/defining-and-using-a-variable-in-batch-file)