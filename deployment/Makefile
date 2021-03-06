# No spaces in STACK_NAME
export STACK_NAME := ${USER}
export EC2_KEY := geotrellis-cluster
export EC2_KEY_FILE := "${HOME}/${EC2_KEY}.pem"
export AWS_DEFAULT_REGION := us-east-1
export S3_URI := s3://geotrellis-test/ca/${USER}
export SUBNET_ID := subnet-c5fefdb1

export MASTER_INSTANCE := m3.xlarge
export MASTER_PRICE := 0.5

# export WORKER_INSTANCE := m3.xlarge
# export CPUS_PER_WORKER := 4

export WORKER_INSTANCE := m3.2xlarge
export CPUS_PER_WORKER := 8

export WORKER_PRICE := 0.5
export WORKER_COUNT := 5

export DRIVER_MEMORY := 4200M
export DRIVER_CORES := 2
export EXECUTOR_MEMORY := 4200M
export EXECUTOR_CORES := 2
export YARN_OVERHEAD := 700

# Docker image of benchmarking service
export SERVICE_TAG := eac43
export SERVICE_IMG := quay.io/geotrellis/comparative-analysis-query-server:${SERVICE_TAG}

SPARK_PARAMS := --driver-memory ${DRIVER_MEMORY} \
--driver-cores ${DRIVER_CORES} \
--executor-memory ${EXECUTOR_MEMORY} \
--executor-cores ${EXECUTOR_CORES} \
--conf spark.dynamicAllocation.enabled=true \
--conf spark.yarn.executor.memoryOverhead=${YARN_OVERHEAD} \
--conf spark.yarn.driver.memoryOverhead=${YARN_OVERHEAD}

SPARK_OPTIONS := --master yarn \
--deploy-mode cluster ${SPARK_PARAMS}

GEODOCKER_BOOTSTRAP=s3://geotrellis-test/geodocker/accumulo-0.2/bootstrap-geodocker-accumulo.sh

EMR_IP=$(shell $(warning "Waiting for emr cluster ${1}") \
  aws emr wait cluster-running --cluster-id ${1} 1>&2 \
	&& aws emr describe-cluster --output json  --cluster-id ${1} | jq -r '.Cluster.MasterPublicDnsName')

define EMR_CREATE_CLUSTER
	$(warning "Creating ${1} cluster")
	aws emr create-cluster --name "CA ${STACK_NAME} ${1}" \
--release-label emr-5.0.0 \
--output text \
--use-default-roles \
--log-uri ${S3_URI}/logs \
--ec2-attributes KeyName=${EC2_KEY},SubnetId=${SUBNET_ID} \
--applications Name=Hadoop Name=Zookeeper Name=Spark Name=Zeppelin Name=Ganglia \
--instance-groups \
Name=Master,InstanceCount=1,InstanceGroupType=MASTER,InstanceType=${MASTER_INSTANCE} \
Name=Workers,InstanceCount=${WORKER_COUNT},InstanceGroupType=CORE,InstanceType=${WORKER_INSTANCE} \
--bootstrap-actions Name=bootstrap-${1},Path=${GEODOCKER_BOOTSTRAP},Args=[-i=${2},-n=gis,-p=secret,${ACCUMULO_ARGS}] \
| tee ${STACK_NAME}-${1}-cluster-id.txt
endef

ACCUMULO_ARGS:=-e=MASTER_XMX=3G,-e=TSERVER_XMX=24G,-e=TSERVER_CACHE_DATA_SIZE=16G,-e=TSERVER_CACHE_INDEX_SIZE=4G

GEOWAVE_CLUSTER_ID=$(shell cat ${STACK_NAME}-geowave-cluster-id.txt)
GEOMESA_CLUSTER_ID=$(shell cat ${STACK_NAME}-geomesa-cluster-id.txt)

GEOWAVE_ZOOKEEPER=$(call EMR_IP,${GEOWAVE_CLUSTER_ID})
GEOMESA_ZOOKEEPER=$(call EMR_IP,${GEOMESA_CLUSTER_ID})

${STACK_NAME}-geomesa-cluster-id.txt:
	$(call EMR_CREATE_CLUSTER,geomesa,quay.io/geodocker/accumulo-geomesa:0.2)

${STACK_NAME}-geowave-cluster-id.txt:
	$(call EMR_CREATE_CLUSTER,geowave,quay.io/geodocker/accumulo-geowave:0.2)

rwildcard=$(foreach d,$(wildcard $1*),$(call rwildcard,$d/,$2) $(filter $(subst *,%,$2),$d))

deploy-clusters: ${STACK_NAME}-geomesa-cluster-id.txt ${STACK_NAME}-geowave-cluster-id.txt
	@echo Clusters Created

APP_NAME:=${STACK_NAME}

deploy: ${STACK_NAME}-geomesa-cluster-id.txt ${STACK_NAME}-geowave-cluster-id.txt
	terraform apply \
-state="${APP_NAME}.tfstate" \
-var 'stack_name=${APP_NAME}' \
-var 'ec2_key=${EC2_KEY}' \
-var 'subnet_id=${SUBNET_ID}' \
-var 'service_image=${SERVICE_IMG}' \
-var 'geomesa_zookeeper=${GEOMESA_ZOOKEEPER}' \
-var 'geowave_zookeeper=${GEOWAVE_ZOOKEEPER}' \
-var 'geomesa_cluster_id=${GEOMESA_CLUSTER_ID}' \
-var 'geowave_cluster_id=${GEOWAVE_CLUSTER_ID}'

destroy-service:
	terraform destroy -force \
-state="${APP_NAME}.tfstate" \
-var 'stack_name=${APP_NAME}' \
-var 'ec2_key=${EC2_KEY}' \
-var 'subnet_id=${SUBNET_ID}' \
-var 'service_image=NA' \
-var 'geomesa_zookeeper=NA' \
-var 'geowave_zookeeper=NA' \
-var 'geomesa_cluster_id=NA' \
-var 'geowave_cluster_id=NA'


destroy: save-metrics  destroy-service
	aws emr terminate-clusters --cluster-ids ${GEOMESA_CLUSTER_ID} ${GEOWAVE_CLUSTER_ID}
	@rm -f ${STACK_NAME}*-cluster-id.txt

ingest-synthetic-data-gm: ASSEMBLY_URI=$(shell ./stage-assembly.sh ../synthetic-data mesaPoke ${S3_URI}/jars)
ingest-synthetic-data-gm:
	@if [ -z "${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi
	./add-steps.sh ${GEOMESA_CLUSTER_ID} "Ingest Synthetic" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.geomesa.MesaPoke ${ASSEMBLY_URI} \
-i=gis -z=${GEOMESA_ZOOKEEPER} -u=root -p=secret -t=geomesa.test \
point,100,uniform:-180:180,uniform:-90:90,fixed:0,100


ingest-synthetic-data-gw: ASSEMBLY_URI=$(shell ./stage-assembly.sh ../synthetic-data wavePoke ${S3_URI}/jars)
ingest-synthetic-data-gw:
	@if [ -z "${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi
	./add-steps.sh ${GEOWAVE_CLUSTER_ID} "Ingest Synthetic" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.geowave.WavePoke ${ASSEMBLY_URI} \
-i=gis -z=${GEOWAVE_ZOOKEEPER} -u=root -p=secret -t=geowave.test --index=space \
point,100,uniform:-180:180,uniform:-90:90,fixed:0,100

include years.mk

ingest-synthetic-cities-gm: export ASSEMBLY_URI=$(shell ./stage-assembly.sh ../synthetic-data mesaPoke ${S3_URI}/jars)
ingest-synthetic-cities-gm:
	@if [ -z "$${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi

	./add-steps.sh ${GEOMESA_CLUSTER_ID} "Ingest Synthetic Background" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.geomesa.MesaPoke $${ASSEMBLY_URI} \
-i=gis -z=${GEOMESA_ZOOKEEPER} -u=root -p=secret -t=geomesa.cities \
point,90,uniform:-180:180,uniform:-90:90,uniform:${Y_1980_START}:${Y_2015_END},100000

	./add-steps.sh ${GEOMESA_CLUSTER_ID} "Ingest Synthetic Cities" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.geomesa.MesaCities $${ASSEMBLY_URI} \
-i=gis -z=${GEOMESA_ZOOKEEPER} -u=root -p=secret -t=geomesa.cities \
point,9,uniform:${Y_1980_START}:${Y_2015_END},100000 \
point,9,uniform:${Y_1985_START}:${Y_2015_END},100000 \
point,9,uniform:${Y_1990_START}:${Y_2015_END},100000 \
point,9,uniform:${Y_1995_START}:${Y_2015_END},100000 \
point,9,uniform:${Y_2000_START}:${Y_2015_END},100000 \
point,9,uniform:${Y_2005_START}:${Y_2015_END},100000 \
point,9,uniform:${Y_2010_START}:${Y_2015_END},100000


ingest-synthetic-cities-gw: export ASSEMBLY_URI=$(shell ./stage-assembly.sh ../synthetic-data wavePoke ${S3_URI}/jars)
ingest-synthetic-cities-gw:
	@if [ -z "$${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi

	./add-steps.sh ${GEOWAVE_CLUSTER_ID} "Ingest Synthetic Background" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.geowave.WavePoke $${ASSEMBLY_URI} \
-i=gis -z=${GEOWAVE_ZOOKEEPER} -u=root -p=secret -t=geowave.cities --index=spacetime \
point,10,uniform:-180:180,uniform:-90:90,uniform:${Y_1980_START}:${Y_2015_END},100000

	./add-steps.sh ${GEOWAVE_CLUSTER_ID} "Ingest Synthetic Cities" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.geowave.WaveCities $${ASSEMBLY_URI} \
-i=gis -z=${GEOWAVE_ZOOKEEPER} -u=root -p=secret -t=geowave.cities --index=spacetime \
point,1,uniform:${Y_1980_START}:${Y_2015_END},100000 \
point,1,uniform:${Y_1985_START}:${Y_2015_END},100000 \
point,1,uniform:${Y_1990_START}:${Y_2015_END},100000 \
point,1,uniform:${Y_1995_START}:${Y_2015_END},100000 \
point,1,uniform:${Y_2000_START}:${Y_2015_END},100000 \
point,1,uniform:${Y_2005_START}:${Y_2015_END},100000 \
point,1,uniform:${Y_2010_START}:${Y_2015_END},100000


ingest-geolife-gm: ASSEMBLY_URI=$(shell ./stage-assembly.sh ../empirical-data geomesa ${S3_URI}/jars)
ingest-geolife-gm:
	@if [ -z "${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi
	./add-steps.sh ${GEOMESA_CLUSTER_ID} "Ingest CSV" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.ingest.geomesa.Main ${ASSEMBLY_URI} \
csv -i gis -z ${GEOMESA_ZOOKEEPER} -u root \
-p secret -t geomesa.geolife -e plt -s , -d 6 \
--codec 'the_geom=point($$2,$$1),height=$$4,timestamp=date({yyyy-MM-ddHH:mm:ss},concat($$6,$$7))' \
--featurename gmtrajectory \
geotrellis-sample-datasets geolife/

# --numPartitions ${NUMPARTITIONS} --partitionStrategy HASH
# --numSplits ${NUMPARTITIONS}
ingest-geolife-gw: ASSEMBLY_URI=$(shell ./stage-assembly.sh ../empirical-data geowave ${S3_URI}/jars)
ingest-geolife-gw: NUMPARTITIONS=$(shell expr ${WORKER_COUNT} \* ${CPUS_PER_WORKER})
ingest-geolife-gw:
	@if [ -z "${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi
	./add-steps.sh ${GEOWAVE_CLUSTER_ID} "Ingest CSV" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.ingest.geowave.Main ${ASSEMBLY_URI} \
csv -i gis -z ${GEOWAVE_ZOOKEEPER} -u root \
-p secret -t geowave.geolife -e plt -s , -d 6 \
--temporal --point \
--codec 'the_geom=point($$2,$$1),height=$$4,timestamp=date({yyyy-MM-ddHH:mm:ss},concat($$6,$$7))' \
--featurename gwtrajectory \
geotrellis-sample-datasets geolife/

ingest-generated-tracks-gm: export ASSEMBLY_URI=$(shell ./stage-assembly.sh ../empirical-data geomesa ${S3_URI}/jars)
ingest-generated-tracks-gm:
	@if [ -z "$${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi
	./add-steps.sh ${GEOMESA_CLUSTER_ID} "Ingest Generated Tracks" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.ingest.geomesa.Main  $${ASSEMBLY_URI} \
avro -i gis -z ${GEOMESA_ZOOKEEPER} -u root \
-p secret -t geomesa.tracks --inputPartitionSize 1 \
--period year \
--featurename generated-tracks \
geotrellis-sample-datasets generated-tracks/

ingest-generated-tracks-gw: export ASSEMBLY_URI=$(shell ./stage-assembly.sh ../empirical-data geowave ${S3_URI}/jars)
ingest-generated-tracks-gw:
	@if [ -z "$${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi
	./add-steps.sh ${GEOWAVE_CLUSTER_ID} "Ingest Generated Tracks" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.ingest.geowave.Main  $${ASSEMBLY_URI} \
avro -i gis -z ${GEOWAVE_ZOOKEEPER} -u root -p secret -t geowave.tracks \
--featurename generated-tracks --inputPartitionSize 1 \
--temporal --period year --numPartitions 4 --partitionStrategy ROUND_ROBIN \
geotrellis-sample-datasets generated-tracks/

ingest-vienna-gm: export ASSEMBLY_URI=$(shell ./stage-assembly.sh ../empirical-data geomesa ${S3_URI}/jars)
ingest-vienna-gm:
	@if [ -z "$${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi
	./add-steps.sh ${GEOMESA_CLUSTER_ID} "Ingest Vienna" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.ingest.geomesa.Main  $${ASSEMBLY_URI} \
shapefile -i gis -z ${GEOMESA_ZOOKEEPER} -u root \
-p secret -t geomesa.shptest --featurename vienna-buildings geotrellis-sample-datasets Vienna/buildings/

ingest-vienna-gw: export ASSEMBLY_URI=$(shell ./stage-assembly.sh ../empirical-data geowave ${S3_URI}/jars)
ingest-vienna-gw:
	@if [ -z "$${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi
	./add-steps.sh ${GEOWAVE_CLUSTER_ID} "Ingest Vienna" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.ingest.geowave.Main  $${ASSEMBLY_URI} \
shapefile -i gis -z ${GEOWAVE_ZOOKEEPER} -u root \
-p secret -t geowave.shptest --featurename vienna-buildings geotrellis-sample-datasets Vienna/buildings/

ingest-vienna-translate-gm: export ASSEMBLY_URI=$(shell ./stage-assembly.sh ../empirical-data geomesa ${S3_URI}/jars)
ingest-vienna-translate-gm:
	@if [ -z "$${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi
	./add-steps.sh ${GEOMESA_CLUSTER_ID} "Ingest Vienna Translated" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.ingest.geomesa.Main  $${ASSEMBLY_URI} \
shapefile -i gis -z ${GEOMESA_ZOOKEEPER} -u root \
-p secret -t geomesa.transvienna --featurename vienna-buildings \
--translationPoints s3://geotrellis-test/ca/ingest/city-points.json --translationOrigin "16.36,48.21" \
geotrellis-sample-datasets Vienna/buildings/


ingest-vienna-translate-gw: export ASSEMBLY_URI=$(shell ./stage-assembly.sh ../empirical-data geowave ${S3_URI}/jars)
ingest-vienna-translate-gw:
	@if [ -z "$${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi
	./add-steps.sh ${GEOWAVE_CLUSTER_ID} "Ingest Vienna Translated" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.ingest.geowave.Main  $${ASSEMBLY_URI} \
shapefile -i gis -z ${GEOWAVE_ZOOKEEPER} -u root \
-p secret -t geowave.transvienna --featurename vienna-buildings \
--translationPoints s3://geotrellis-test/ca/ingest/city-points.json --translationOrigin "16.36,48.21" \
geotrellis-sample-datasets Vienna/buildings/

ingest-vienna-translate: ingest-vienna-translate-gw ingest-vienna-translate-gm

ingest-tracks-translate-gm: export ASSEMBLY_URI=$(shell ./stage-assembly.sh ../empirical-data geomesa ${S3_URI}/jars)
ingest-tracks-translate-gm:
	@if [ -z "$${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi
	./add-steps.sh ${GEOMESA_CLUSTER_ID} "Ingest Generated Tracks Translation" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.ingest.geomesa.Main  $${ASSEMBLY_URI} \
avro -i gis -z ${GEOMESA_ZOOKEEPER} -u root \
-p secret -t geomesa.tracks \
--featurename generated-tracks \
--translationPoints s3://geotrellis-test/ca/ingest/slight-offsets.json --translationOrigin "0.0,0.0" \
geotrellis-sample-datasets generated-tracks/

ingest-tracks-translate-gw: ASSEMBLY_URI=$(shell ./stage-assembly.sh ../empirical-data geowave ${S3_URI}/jars)
ingest-tracks-translate-gw:
	@if [ -z "${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi
	./add-steps.sh ${GEOWAVE_CLUSTER_ID} "Ingest Generated Tracks Translation" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.ingest.geowave.Main  ${ASSEMBLY_URI} \
avro -i gis -z ${GEOWAVE_ZOOKEEPER} -u root \
-p secret -t geowave.tracks \
--featurename generated-tracks \
--temporal \
--translationPoints s3://geotrellis-test/ca/ingest/slight-offsets.json --translationOrigin "0.0,0.0" \
geotrellis-sample-datasets generated-tracks/

ingest-tracks-translate: ingest-tracks-translate-gw ingest-tracks-translate-gm

ingest-gdelt-gm: ASSEMBLY_URI=$(shell ./stage-assembly.sh ../empirical-data geomesa ${S3_URI}/jars)
ingest-gdelt-gm:
	@if [ -z "${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi
	./add-steps.sh ${GEOMESA_CLUSTER_ID} "Ingest GDELT GeoMesa" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.ingest.geomesa.GdeltIngest ${ASSEMBLY_URI} ${GEOMESA_ZOOKEEPER}

ingest-gdelt-gw: ASSEMBLY_URI=$(shell ./stage-assembly.sh ../empirical-data geowave ${S3_URI}/jars)
ingest-gdelt-gw:
	@if [ -z "${ASSEMBLY_URI}" ]; then echo "Assembly failed" && exit 1; fi
	./add-steps.sh ${GEOWAVE_CLUSTER_ID} "Ingest GDELT GeoWave" \
spark-submit ${SPARK_OPTIONS} --class com.azavea.ingest.geowave.GdeltIngest ${ASSEMBLY_URI} ${GEOWAVE_ZOOKEEPER}

ingest-gdelt: ingest-gdelt-gw ingest-gdelt-gm

../query-server/explore/target/scala-2.11/explore-assembly-0.0.1.jar: $(call rwildcard, $(call rwildcard, server, *.scala) build.sbt, *.scala) ../query-server/explore/build.sbt
	cd ../query-server/ && ./sbt "project explore" assembly

explore: ../query-server/explore/target/scala-2.11/explore-assembly-0.0.1.jar
	echo "GM_CLUSTER_ID=${GEOMESA_CLUSTER_ID} GW_CLUSTER_ID=${GEOWAVE_CLUSTER_ID} GM_ZK=${GEOMESA_ZOOKEEPER} GW_ZK=${GEOWAVE_ZOOKEEPER} GM_USER=root GW_USER=root GM_PASS=secret GW_PASS=secret GM_INSTANCE=gis GW_INSTANCE=gis spark-shell --master yarn --deploy-mode client ${SPARK_PARAMS} --jars explore-assembly-0.0.1.jar" > run-explore-shell.sh
	chmod a+x run-explore-shell.sh
	scp -i ${EC2_KEY_FILE} ../query-server/explore/target/scala-2.11/explore-assembly-0.0.1.jar hadoop@${GEOMESA_ZOOKEEPER}:/home/hadoop
	scp -i ${EC2_KEY_FILE} run-explore-shell.sh hadoop@${GEOMESA_ZOOKEEPER}:/home/hadoop
	ssh -i ${EC2_KEY_FILE} hadoop@${GEOMESA_ZOOKEEPER}

save-metrics:
	@mkdir -p ../metrics
	@echo Saving GeoMesa metrics...
	@aws emr ssh --cluster-id ${GEOMESA_CLUSTER_ID} --key-pair-file "${EC2_KEY_FILE}" \
		--command "cd /mnt/var/lib/ganglia/rrds && tar czf ~/metrics.tgz j-* && aws s3 cp ~/metrics.tgz ${S3_URI}/metrics/${GEOMESA_CLUSTER_ID}.tgz"

	@echo Save GeoWave metrics...
	@aws emr ssh --cluster-id ${GEOWAVE_CLUSTER_ID} --key-pair-file "${EC2_KEY_FILE}" \
		--command "cd /mnt/var/lib/ganglia/rrds && tar czf ~/metrics.tgz j-* && aws s3 cp ~/metrics.tgz ${S3_URI}/metrics/${GEOWAVE_CLUSTER_ID}.tgz"

fetch-metrics:
	@mkdir -p ../metrics
	@echo Fetching GeoMesa metrics...
	@aws emr ssh --cluster-id ${GEOMESA_CLUSTER_ID} --key-pair-file "${EC2_KEY_FILE}" \
		--command "cd /mnt/var/lib/ganglia/rrds && tar czf ~/metrics.tgz j-*"
	@aws emr get --cluster-id ${GEOMESA_CLUSTER_ID} --key-pair-file "${EC2_KEY_FILE}" --src "~/metrics.tgz" --dest ../metrics/${GEOMESA_CLUSTER_ID}.tgz

	@echo Fetching GeoWave metrics...
	@aws emr ssh --cluster-id ${GEOWAVE_CLUSTER_ID} --key-pair-file "${EC2_KEY_FILE}" \
		--command "cd /mnt/var/lib/ganglia/rrds && tar czf ~/metrics.tgz j-*"
	@aws emr get --cluster-id ${GEOWAVE_CLUSTER_ID} --key-pair-file "${EC2_KEY_FILE}" --src "~/metrics.tgz" --dest ../metrics/${GEOWAVE_CLUSTER_ID}.tgz

proxy-gm:
	aws emr socks --cluster-id ${GEOMESA_CLUSTER_ID} --key-pair-file "${EC2_KEY_FILE}"

proxy-gw:
	aws emr socks --cluster-id ${GEOWAVE_CLUSTER_ID} --key-pair-file "${EC2_KEY_FILE}"
