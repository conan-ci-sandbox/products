import groovy.json.JsonOutput

config_url = "https://github.com/conan-ci-sandbox/settings.git"

artifactory_metadata_repo = "conan-metadata"
conan_develop_repo = "conan-develop"
conan_tmp_repo = "conan-tmp"
artifactory_url = (env.ARTIFACTORY_URL != null) ? "${env.ARTIFACTORY_URL}" : "jfrog.local"

build_order_file = "bo.json"
build_order = null

profiles = [
  "debug-gcc6": "conanio/gcc6",	
  "release-gcc6": "conanio/gcc6"	
]

build_result = [:]

user = "mycompany"
channel = "stable"

products = ["App/1.0@mycompany/stable", "App2/1.0@mycompany/stable"]

affected_products = []

def promote_with_lockfile(lockfile_json, source_repo, target_repo, additional_references=[]) {
  // additional_references are for the case, like in this workflow
  // that we have a lockfile with some nodes marked as build but they
  // are the result of creating a new revision of another lib in the pipeline
  // that triggered this one. That library is not going to be in the lockfile
  // marked as build so maybe we want to add some references to the promotion
  def references_to_copy = []
  def nodes = lockfile_json['graph_lock'].nodes
  nodes.each { id, node_info ->
    if (node_info.modified) {
      references_to_copy.add(node_info.pref)
    }
    else {
      additional_references.each { reference ->
        if (node_info.pref.startsWith(reference)) {
          references_to_copy.add(node_info.pref)
        }
      }
    }
  }
  references_to_copy.unique()
  // now move all the package references that were build
  references_to_copy.each { pref ->
    // path: repo/user/name/version/channel/rrev/package/pkgid/prev/conan_package.tgz
    echo "copy ${pref} from ${source_repo} to ${target_repo}"
    name_version = pref.split("#")[0].split("@")[0]
    rrev = pref.split("#")[1].split(":")[0]
    def pkgid = pref.split("#")[1].split(":")[1]
    def prev = pref.split("#")[2]
    echo "copy name: ${name_version} to ${target_repo}"
    echo "copy rrev: ${rrev} to ${target_repo}"
    echo "copy pkgid: ${pkgid} to ${target_repo}"
    echo "copy prev: ${prev} to ${target_repo}"
    withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
      sh "curl -u\"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -XPOST \"http://${artifactory_url}:8081/artifactory/api/copy/${source_repo}/${user}/${name_version}/${channel}/${rrev}/export?to=${target_repo}/${user}/${name_version}/${channel}/${rrev}\""
      // avoid copying again the same package revision
      get_folder_out = sh (script: "curl -u\"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -XGET \"http://${artifactory_url}:8081/artifactory/api/storage/${target_repo}/${user}/${name_version}/${channel}/${rrev}/package/${pkgid}/${prev}\"", returnStdout: true)
      echo "${get_folder_out}"
      if (get_folder_out.contains("errors")) {
        echo "ERROR 404"
        sh "curl -u\"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -XPOST \"http://${artifactory_url}:8081/artifactory/api/copy/${source_repo}/${user}/${name_version}/${channel}/${rrev}/package/${pkgid}/${prev}?to=${target_repo}/${user}/${name_version}/${channel}/${rrev}/package/${pkgid}/${prev}\""
      }
      else {
        echo "skip copy"        
      }
    } 
  }
}

def calc_lockfiles(product, docker_image) {
  return {
    stage("Calc lockfiles") {
      node {
        docker.image(docker_image).inside("--net=host") {
          echo "Inside docker image: ${docker_image}"
          withEnv(["CONAN_USER_HOME=${env.WORKSPACE}/conan_home"]) {
            try {
              stage("Configure conan") {
                sh "conan config install ${config_url}"
                withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                    sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_develop_repo} ${ARTIFACTORY_USER}"
                    sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_tmp_repo} ${ARTIFACTORY_USER}"
                }
              }
              // create the graph lock for the latest versions of dependencies in develop repo and with the
              // latest revision of the library that trigered this pipeline
              sh "conan download ${params.reference} -r ${conan_tmp_repo} --recipe"
              def name = product.split("/")[0]
              def profiles_build_order = [:]
              stage("Calculate lockfiles for building ${name} with ${params.reference}") {
                profiles.each { profile, _ ->
                  echo "calc lockfile for profile: ${profile}"
                  def lockfile = "${name}-${profile}.lock"
                  // install just the recipe for the new revision of the lib and then resolve graph with develop repo
                  // we don't want to bring binaries at this moment, this could be just a agent that
                  // then sends lockfiles to other nodes to build them
                  sh "conan graph lock ${product} --profile=${profile} --lockfile=${lockfile} -r ${conan_develop_repo}"
                  stash name: lockfile, includes: lockfile 
                  def build_order_file = "${name}-${profile}.json"             
                  sh "conan graph build-order ${lockfile} --json=${build_order_file} --build missing"
                  def build_order = readJSON(file: build_order_file)
                  profiles_build_order[lockfile] = build_order
                }              
              }

              println profiles_build_order
              stage("Consolidate build order") {

                def libs_to_build = []
                def keep_going = true

                while (keep_going == true) {
                    keep_going = false
                    def build_map = [:]
                    profiles_build_order.each { profile, build_order ->
                        def inverse_build_order = build_order.reverse()
                        if (inverse_build_order.size()>0) {
                            keep_going = true
                            for (references_list in inverse_build_order) {
                                references_list.each { index_reference ->
                                    def lib_name = index_reference[1].split("/")[0]
                                    if (!build_map.containsKey(lib_name)) {
                                        build_map[lib_name] = [profile]
                                    }
                                    else {
                                        build_map[lib_name].add(profile)
                                    }
                                }
                                break
                            }
                            println build_order
                            build_order.removeAt(build_order.size()-1)
                            println build_order
                            if (build_order.size()>0) {
                                keep_going = true
                            }
                        }
                    }
                    if (!build_map.isEmpty()) {
                        libs_to_build.add(build_map)
                    }
                }

                def consolidated_build_order = libs_to_build.reverse()
                println consolidated_build_order

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

def build_ref_with_lockfile(reference, lockfile, profile, upload_ref) {
  return {
    stage("Build: ${reference} - ${profile}") {
      unstash lockfile
      def actual_reference_name = reference.split("/")[0]
      def recipe_reference_with_revision = reference.split(":")[0]
      def actual_reference = reference.split("#")[0]
      echo("Build ${actual_reference_name}")
      sh "cp ${lockfile} conan.lock"
      sh "conan install ${actual_reference} --build ${actual_reference_name} --lockfile conan.lock"
      sh "mv conan.lock ${actual_reference_name}-${profile}.lock"
      stash name: "${actual_reference_name}-${profile}.lock", includes: "${actual_reference_name}-${profile}.lock"
      stage ("Upload reference ${actual_reference}-${profile} to ${conan_tmp_repo}") {
        if (upload_ref) {
          sh "conan upload ${actual_reference} --all -r ${conan_tmp_repo} --confirm"
        }
      }
    }
  }
}

def build_with_lockfiles() {
  return {
    stage(profile) {
      node {
        docker.image(docker_image).inside("--net=host") {
          echo "Inside docker image: ${docker_image}"
          withEnv(["CONAN_USER_HOME=${env.WORKSPACE}/${profile}/conan_home"]) {
            try {
              stage("Configure conan") {
                sh "conan config install ${config_url}"
                withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                    sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_develop_repo} ${ARTIFACTORY_USER}"
                    sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_tmp_repo} ${ARTIFACTORY_USER}"
                }
              }
              // create the graph lock for the latest versions of dependencies in develop repo and with the
              // latest revision of the library that trigered this pipeline
              def lockfile = "${profile}.lock"
              unstash lockfile
              def bo_file = "build_order.json"
              def lock_contents = [:]
              sh "conan graph build-order ${lockfile} --json=${bo_file} --build missing"
              build_order = readJSON(file: bo_file)
              stage("Launch build: ${product}")
              {
                stash name: lockfile, includes: lockfile
                build_order.each { references_list ->
                  def stage_jobs = references_list.each { index_reference ->
                    def lib_name = index_reference[1].split("/")[0]
                    def lib_name_profile = "${lib_name}-${profile}.lock"
                    // here, in the real world, one should invoke the actual lib pipeline, somemthing like:
                    // build(job: "../lib/develop", propagate: true, wait: true, parameters...
                    def upload_ref = (params.library_branch == "develop") ? true : false
                    build_ref_with_lockfile(index_reference[1], lockfile, profile, upload_ref).call()
                    unstash lib_name_profile
                    sh "conan graph update-lock ${lockfile} ${lib_name_profile}"
                    stash name: lockfile, includes: lockfile
                  }
                }                  
                lock_contents = readJSON(file: lockfile)
                sh "cat ${lockfile}"
              }

              return lock_contents              
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

pipeline {
  agent none

  parameters {
    string(name: 'reference',)
    string(name: 'library_branch',)
  }

  stages {
    stage('Build affected products') {
      agent any
      steps {
        script {   
          products_build_result = products.collectEntries { product ->
            def name = product.split("/")[0]
  
            stage("Calc lockfiles for ${product}") {
              echo "Calc lockfiles for '${product}'"
              echo " - for changes in '${params.reference}'"
              calc_lockfiles(product, "conanio/gcc6").call()
            }

            profiles.each { profile, docker_image ->
              def lockfile = "${name}-${profile}.lock"
              unstash lockfile
              sh "cat ${lockfile}"
            }

            stage("Build ${product}") {
              echo "Building product '${product}'"
              echo " - for changes in '${params.reference}'"
              build_result = parallel profiles.collectEntries { profile, docker_image ->
                ["${profile}": get_build_stages(product, profile, docker_image)]
              }              
            }
            ["${product}": build_result]
          }
          println products_build_result
        }
      }
    }

    stage("Upload to develop repo") {
      when {expression { return params.library_branch == "develop" }}
      steps {
        script {
          docker.image("conanio/gcc6").inside("--net=host") {
            // promote libraries to develop
            withEnv(["CONAN_USER_HOME=${env.WORKSPACE}/conan_cache"]) {
              try {
                sh "conan config install ${config_url}"
                withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                    sh "conan remote add ${conan_develop_repo} http://${artifactory_url}:8081/artifactory/api/conan/${conan_develop_repo}" // the namme of the repo is the same that the arttifactory key
                    sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_develop_repo} ${ARTIFACTORY_USER}"
                    sh "conan remote add ${conan_tmp_repo} http://${artifactory_url}:8081/artifactory/api/conan/${conan_tmp_repo}" // the namme of the repo is the same that the arttifactory key
                    sh "conan user -p ${ARTIFACTORY_PASSWORD} -r ${conan_tmp_repo} ${ARTIFACTORY_USER}"
                }

                // promote using each of the lockfiles
                products_build_result.each { product, result ->
                  result.each { profile, lockfile ->
                    if (lockfile.size()>0) {
                      stage("Promote ${profile} binaries to conan-develop") {
                        promote_with_lockfile(lockfile, conan_tmp_repo, conan_develop_repo, ["${params.reference}"])
                      }
                      stage("Upload lockfile: ${profile} - ${product}") {
                        def name = product.split("/")[0]
                        def lockfile_name = "${name}-${profile}.lock"
                        writeJSON file: "${lockfile_name}", json: lockfile
                        def lockfile_path = "/${artifactory_metadata_repo}/${env.JOB_NAME}/${env.BUILD_NUMBER}/${product}/${profile}/${lockfile_name}"
                        def base_url = "http://${artifactory_url}:8081/artifactory"
                        def version = product.split("/")[1].split("@")[0]
                        def properties = "?properties=build.name=${env.JOB_NAME}%7Cbuild.number=${env.BUILD_NUMBER}%7Cprofile=${profile}%7Cname=${name}%7Cversion=${version}"
                        withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                            // upload the lockfile
                            sh "curl --user \"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -X PUT ${base_url}${lockfile_path} -T ${lockfile_name}"
                            // set properties in Artifactory for the file
                            sh "curl --user \"\${ARTIFACTORY_USER}\":\"\${ARTIFACTORY_PASSWORD}\" -X PUT ${base_url}/api/storage${lockfile_path}${properties}"
                        }                                
                      }                            
                    }
                  }
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
