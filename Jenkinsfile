#!/usr/bin/env groovy

def installBuildRequirements(){
	def nodeHome = tool 'nodejs-lts-20'
	env.PATH="${env.PATH}:${nodeHome}/bin"
	sh "npm install --global yarn"
	sh "npm install --global @cyclonedx/cdxgen"
}

node('rhel8'){

	stage 'Checkout vscode-kaoto code'
	deleteDir()
	git branch: 'main', url: 'https://github.com/KaotoIO/vscode-kaoto.git'

	stage 'install vscode-kaoto build requirements'
	installBuildRequirements()

	stage 'Build vscode-kaoto'
	sh "yarn"
	sh "yarn build:dev"
	sh "yarn build:prod"

	stage('Test') {
		wrap([$class: 'Xvnc']) {
			sh "yarn test:it"
		}
	}

	stage 'Package vscode-kaoto'
	def packageJson = readJSON file: 'package.json'
	sh "yarn vsce package --no-dependencies --yarn -o vscode-kaoto-${packageJson.version}-${env.BUILD_NUMBER}.vsix"

	stage 'Upload vscode-kaoto to staging'
	def vsix = findFiles(glob: '**.vsix')
	sh "sftp -C ${UPLOAD_LOCATION}/snapshots/vscode-kaoto/ <<< \$'put -p \"${vsix[0].path}\"'"
	stash name:'vsix', includes:vsix[0].path

	stage 'Generate SBOM'
	sh "cdxgen -o manifest.json"
	archive includes:"manifest.json"
}

node('rhel8'){
	if(publishToMarketPlace.equals('true')){
		timeout(time:5, unit:'DAYS') {
			input message:'Approve deployment?', submitter: 'apupier, djelinek, mdinizde, ricmarti, toigaras'
		}

		stage 'Publish to Marketplaces'
		unstash 'vsix';
		def vsix = findFiles(glob: '**.vsix')
		// VS Code Marketplace
		withCredentials([[$class: 'StringBinding', credentialsId: 'vscode_java_marketplace', variable: 'TOKEN']]) {
			sh 'yarn vsce publish -p ${TOKEN} --packagePath' + " ${vsix[0].path}"
		}

		// Open-vsx Marketplace
		sh "npm install -g ovsx"
		withCredentials([[$class: 'StringBinding', credentialsId: 'open-vsx-access-token', variable: 'OVSX_TOKEN']]) {
			sh 'ovsx publish -p ${OVSX_TOKEN}' + " ${vsix[0].path}"
		}
		archive includes:"**.vsix"

		stage ('Promote the build to stable') {
			vsix = findFiles(glob: '**.vsix')
			sh "sftp -C ${UPLOAD_LOCATION}/stable/vscode-kaoto/ <<< \$'put -p \"${vsix[0].path}\"'"
		}
	}
}
