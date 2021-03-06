#!/usr/bin/env groovy

properties([
buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '30', numToKeepStr: '5')), 
[$class: 'GithubProjectProperty', displayName: '', projectUrlStr: 'https://gitenterprise.xilinx.com/SDx-Hub/apps/'], 
pipelineTriggers([[$class: 'GitHubPushTrigger']]),
disableConcurrentBuilds()
])

// Days between clean builds
days = 10

devices = []
devices += ['xilinx:adm-pcie-7v3:1ddr:3.0']
devices += ['xilinx:xil-accel-rd-ku115:4ddr-xpr:3.3']
devices += ['xilinx:adm-pcie-ku3:2ddr-xpr:3.3']

version = '2016.4'

def setupExample(dir, workdir) {
	return { ->
		sh """#!/bin/bash -e

cd ${workdir}

. /tools/local/bin/modinit.sh > /dev/null 2>&1
module use.own /proj/picasso/modulefiles

module add vivado/${version}_daily
module add vivado_hls/${version}_daily
module add sdaccel/${version}_daily
module add opencv/vivado_hls

cd ${dir}

echo
echo "-----------------------------------------------"
echo "PWD: \$(pwd)"
echo "-----------------------------------------------"
echo

rsync -rL \$XILINX_SDX/Vivado_HLS/lnx64/tools/opencv/ lib/

make clean exe

"""
	}
}

def buildExample(target, dir, device, workdir) {
	return { ->
		if ( target == "sw_emu" ) {
			cores = 1
			queue = "medium"
		} else {
			cores = 8
			queue = "long"
		}

		sh """#!/bin/bash -e

cd ${workdir}

. /tools/local/bin/modinit.sh > /dev/null 2>&1
module use.own /proj/picasso/modulefiles

module add vivado/${version}_daily
module add vivado_hls/${version}_daily
module add sdaccel/${version}_daily
module add opencv/vivado_hls
module add lsf

cd ${dir}

echo
echo "-----------------------------------------------"
echo "PWD: \$(pwd)"
echo "-----------------------------------------------"
echo

# Check if a rebuild is necessary
set -x
set +e
make -q TARGETS=${target} DEVICES=\"${device}\" all
rc=\$?
set -e

# if rebuild required then use LSF
if [[ \$rc != 0 ]]; then
bsub -I -q ${queue} -R "osdistro=rhel && osver==ws6" -n ${cores} -R "span[ptile=${cores}]" -J "\$(basename ${dir})-${target}" <<EOF
make TARGETS=${target} DEVICES=\"${device}\" all
EOF
fi
set +x
"""
	}
}

def dirsafe(device) {
	return device.replaceAll(":", "_").replaceAll("\\.", "_")
}

def runExample(target, dir, device, workdir) {
	return { ->
		devdir = dirsafe(device)
		/* Node is here to prevent too much strain on Nimbix by rate limiting
		 * to the number of job slots */
		node("rhel6 && xsjrdevl && !xsjrdevl110") {
			retry(3) {
				sh """#!/bin/bash -e

cd ${workdir}

. /tools/local/bin/modinit.sh > /dev/null 2>&1
module use.own /proj/picasso/modulefiles

module add vivado/${version}_daily
module add vivado_hls/${version}_daily
module add sdaccel/${version}_daily
module add opencv/vivado_hls

module add proxy
module add lftp

cd ${dir}

echo
echo "-----------------------------------------------"
echo "PWD: \$(pwd)"
echo "-----------------------------------------------"
echo

export PYTHONUNBUFFERED=true

rm -rf \"out/${target}_${devdir}\" && mkdir -p out

make TARGETS=${target} DEVICES=\"${device}\" NIMBIXFLAGS=\"--out out/${target}_${devdir} --queue_timeout=480\" check

"""
			}
		}
	}
}

timestamps {
// Always build on the same host so that the workspace is reused
node('rhel6 && xsjrdevl && xsjrdevl110') {
try {

	stage("checkout") {
		checkout scm
	}

	stage("clean") {
		try {
			lastclean = readFile('lastclean.dat')
		} catch (e) {
			lastclean = "01/01/1970"
		} finally {
			lastcleanDate = new Date(lastclean)
			echo "Last Clean Build on ${lastcleanDate}"
		}
        
		def date = new Date()
		echo "Current Build Date is ${date}"

		if(date > lastcleanDate + days) {
			echo "Build too old reseting build area"
			sh 'git clean -xfd'
			// After clean write new lastclean Date to lastclean.dat
			writeFile(file: 'lastclean.dat', text: "${date}")
		}
	}

	stage('pre-check') {
		sh """
. /tools/local/bin/modinit.sh > /dev/null 2>&1
module use.own /proj/picasso/modulefiles

module add vivado/${version}_daily
module add vivado_hls/${version}_daily
module add sdaccel/${version}_daily
module add opencv/vivado_hls

module add proxy

./utility/check_license.sh LICENSE.txt
./utility/check_readme.sh
"""
	}

	workdir = pwd()

	def examples = []

	stage('configure') {
		sh 'git ls-files | grep description.json | sed -e \'s/\\.\\///\' -e \'s/\\/description.json//\' > examples.dat'
		examplesFile = readFile 'examples.dat'
		examples = examplesFile.split('\\n')

		def exSteps = [:]

		for(int i = 0; i < examples.size(); i++) {
			name = "${examples[i]}-setup"
			exSteps[name] = setupExample(examples[i], workdir)
		}

		parallel exSteps
	}

	stage('sw_emu') {
		def swEmuSteps = [:]
		def swEmuRunSteps = [:]

		for(int i = 0; i < examples.size(); i++) {
			for(int j = 0; j < devices.size(); j++) {
				name = "${examples[i]}-${devices[j]}-sw_emu"
				swEmuSteps["${name}-build"]  = buildExample('sw_emu', examples[i], devices[j], workdir)
				swEmuRunSteps["${name}-run"] = runExample(  'sw_emu', examples[i], devices[j], workdir)
			}
		}

		parallel swEmuSteps

		lock("only_one_run_stage_at_a_time") {
			parallel swEmuRunSteps
		}
	}

	stage('hw') {
		def hwSteps = [:]
		def hwRunSteps = [:]

		for(int i = 0; i < examples.size(); i++) {
			for(int j = 0; j < devices.size(); j++) {
				name = "${examples[i]}-${devices[j]}-hw"
				hwSteps["${name}-build"]  = buildExample('hw', examples[i], devices[j], workdir)
				hwRunSteps["${name}-run"] = runExample(  'hw', examples[i], devices[j], workdir)
			}
		}

		parallel hwSteps

		lock("only_one_run_stage_at_a_time") {
			parallel hwRunSteps
		}
	}

} catch (e) {
	currentBuild.result = "FAILED"
	throw e
} finally {
	stage('post-check') {
		step([$class: 'GitHubCommitStatusSetter'])
		step([$class: 'Mailer', notifyEveryUnstableBuild: true, recipients: 'spenserg@xilinx.com', sendToIndividuals: false])
	}
	stage('cleanup') {
		// Cleanup .Xil Files after run
		sh 'find . -name .Xil | xargs rm -rf'
	}
} // try
} // node
} // timestamps


