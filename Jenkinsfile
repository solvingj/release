import groovy.json.JsonOutput

def config_url = "https://github.com/conan-ci-cd-training/settings.git"

def artifactory_metadata_repo = "conan-metadata"
def conan_develop_repo = "conan-develop"
def conan_tmp_repo = "conan-tmp"
def artifactory_url = (env.ARTIFACTORY_URL != null) ? "${env.ARTIFACTORY_URL}" : "jfrog.local"

def build_order_file = "bo.json"
def build_order = null

def profiles = [
  "debug-gcc6": "conanio/gcc6",	
  "release-gcc6": "conanio/gcc6"	
]

def build_result = [:]

def user = "mycompany"
def channel = "stable"

def products = [
  "App/1.0@mycompany/stable": "https://github.com/conan-ci-cd-training/App.git",	
  "App2/1.0@mycompany/stable": "https://github.com/conan-ci-cd-training/App2.git"	
]

def affected_products = []


pipeline {
  agent none

  parameters {
    string(name: 'product',)
    string(name: 'build_name',)
    string(name: 'build_number',)
    string(name: 'profile',)
  }

  def product = (params.product != null) ? "${params.product}" : "App/1.0@mycompany/stable"
  def build_name = (params.build_name != null) ? "${params.build_name}" : "App/develop"
  def build_number = (params.build_number != null) ? "${params.build_number}" : "2"
  def profile = (params.profile != null) ? "${params.profile}" : "release-gcc6"

  stages {
    stage("Create debian package") {
      steps {
        script {
          if (library_branch == "develop") {  
            docker.image("conanio/gcc6").inside("--net=host") {
              // promote libraries to develop
              withEnv(["CONAN_USER_HOME=${env.WORKSPACE}/conan_cache"]) {
                try {
                  sh "conan config install ${config_url}"
                  sh "conan remote add ${conan_develop_repo} http://${artifactory_url}:8081/artifactory/api/conan/${conan_develop_repo}" // the namme of the repo is the same that the arttifactory key
                  withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                      sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_develop_repo} ${ARTIFACTORY_USER}"
                      def lockfile_url = "http://${artifactory_url}:8081/artifactory/${artifactory_metadata_repo}/${build_name}/${build_number}/${product}/${profile}/conan.lock"
                      sh "curl --user \"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -o conan.lock ${lockfile_url}"
                      sh "mkdir deploy && cd deploy && conan install ${product} --lockfile conan.lock -g deploy -r ${conan_develop_repo}"
                      ​def version = product.split("/")[1].split("@")[0]
                      sh "./generateDebianPkg.sh ${version}"
                      // Upload package to artifactory
                      def deb_url = "http://${artifactory_url}:8081/artifactory/app-debian-sit-local/pool/myapp_${version}.deb;deb.distribution=stretch;deb.component=main;deb.architecture=x86-64" 
                      sh "curl --user \"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -X PUT ${deb_url} -T myapp_${version}.deb"
                  }
                }
                finally {
                  deleteDir()
                }
              }
            }
          }
        }
      }
    }
  }
}
