properties([
  [$class: 'JiraProjectProperty'],
  [$class: 'RebuildSettings', autoRebuild: false, rebuildDisabled: false],
  [$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', artifactDaysToKeepStr: '3', artifactNumToKeepStr: '4', daysToKeepStr: '3', numToKeepStr: '5']],

  parameters([
    extendedChoice(
      defaultValue: 'Smoke Pack',
      description: 'Multi select list of stages to be executed during this execution',
      multiSelectDelimiter: ',', name: 'STAGES_TO_BE_EXECUTED',
      quoteValue: false, saveJSONParameterToFile: false,
      type: 'PT_MULTI_SELECT',
      value: 'Full Pack,Smoke Pack,Main Pack,Special Pack, -------------,Stage1,Stage2,Stage3,Stage4,Stage5,Stage6,Stage7',
      visibleItemCount: 12)
  ])
])

jestCommand = "node_modules/.bin/jest --config=jest.server.config.js --reporters='default' --reporters='jest-allure'"
nodejsConfig = [cacheLocationStrategy: workspace(), configId: 'my-ssh-key', nodeJSInstallationName: 'NodeJS13_13_0']

jobPacks = [
 'Smoke Pack' : ["Stage1", "Stage2", "Stage3"],
 'Main Pack' : ["Stage4", "Stage5", "Stage6", "Stage7"],
 'Special Pack': ["Stage8", "Stage9"]
]
parallelSubStages = [
 'Stage6': ["sub stage 61", "sub stage 62", "sub stage 63", "sub stage 64"],
 'Stage5': ["sub stage 51", "sub stage 52", "sub stage 53"],
 'Stage9': ["sub stage 91", "sub stage 92", "sub stage 93", "sub stage 94", "sub stage 95"]
]
multiStepsStages = [
 'Stage7': ["sub stage 71", "sub stage 72", "sub stage 73"],
 'Stage8': ["sub stage 81", "sub stage 82"]
]

def stageEnv(stageName) {
  return ["PATH_TO_TESTS=${stageName}", "LOG_CALLS=false", "LOG_TEST=true", "LOG_DIFF=false", "ALLURE_NO_ANALYTICS=1"]
}

def generateStep(stageName) {
  return catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
    withEnv(stageEnv(stageName.toLowerCase().replace(" ", "-"))) {
      nodejs(nodejsConfig) {
        sh "${jestCommand}"
      }
    }
  }
}

def generateStage(input) {
  def (stageName, packName) = input

   // multi-steps stage
  if (multiStepsStages.containsKey(stageName)) {
    return {
      stage("${packName}: ${stageName}") {
        // for example generateStep('stage5/sub-stage-51')
        //             generateStep('stage5/sub-stage-52')
        //             etc
        multiStepsStages.getAt(stageName).each { stepName -> generateStep(stageName + '/' + stepName) }
      }
    }
  }

  // single step stage
  return {
    stage("${packName}: ${stageName}") {
      timeout(15) {
        generateStep(stageName)
      }
    }
  }
}

def generateExecuteStagesList(String packName) {
  List<String> stagesToBeExecuted
  if (env.STAGES_TO_BE_EXECUTED.contains(packName) || env.STAGES_TO_BE_EXECUTED.contains('Full Pack')) {
    // if pack is selected add all stages in pack
    stagesToBeExecuted = jobPacks[packName]
  }
  else {
    def selectedStages = env.STAGES_TO_BE_EXECUTED.split(',')

    stagesToBeExecuted = jobPacks[packName].findAll { selectedStages.contains(it) }
  }
  // now replace all stages with sub stages (from parallelSubStages) with related sub stages
  parallelSubStages.each {
    multiStage, stages ->
      if (stagesToBeExecuted.contains(multiStage)) {
        stagesToBeExecuted.remove(multiStage) // remove muliStage (for example "Stage3")
        stagesToBeExecuted.addAll(stages.collect{ "${multiStage}/${it}" }) // add multi-stage sub stages (for examples "Stage3/sub-stage-31")
      }
  }
  return stagesToBeExecuted
}

node {

  stage('prepare test run') {
    git branch: '$BRANCH_NAME', changelog: true, url: 'ssh://git@my-super-puper-repository.git'
    nodejs(nodejsConfig) {
      sh "yarn install"
    }
  }

  jobPacks.each {
    packName, stages ->
      List<String> stagesToExecute = generateExecuteStagesList(packName)

      if (!stagesToExecute.isEmpty()) {
        stage(packName) {
          parallel stagesToExecute.collectEntries { ["${it}": generateStage([it, packName])] }
        }
      }
  }

  stage('Declarative: Post Actions') {
    allure includeProperties: false, jdk: 'openjdk-linux-intel64-1.8.0_211-1', reportBuildPolicy: 'ALWAYS', results: [[path: 'allure-results']]
    nodejs(nodejsConfig) {
      sh label: 'clear jest cache', script: "node_modules/.bin/jest --clearCache || true"
    }
    cleanWs()
  }
} 
