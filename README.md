# CONFIGURING AND RUNNING ELASTIC STACK WITH FILEBEAT. 
# EVERYTHING IS RUNNING ON DOCKER.

#Contents of filebeat.yml

filebeat.config:
  modules:
    path: ${path.config}/modules.d/*.yml
    reload.enabled: false

filebeat.autodiscover:
  providers:
    - type: docker
      hints.enabled: true

processors:
- add_cloud_metadata: ~

output.elasticsearch:
  hosts: '${ELASTICSEARCH_HOSTS:127.0.0.1:9200}'



#Run Filebeat

docker run -d \
  --name=filebeat \
  --user=root \
  --volume="$(pwd)/filebeat.docker.yml:/usr/share/filebeat/filebeat.yml:ro" \
  --volume="/var/lib/docker/containers:/var/lib/docker/containers:ro" \
  --volume="/var/run/docker.sock:/var/run/docker.sock:ro" \
  --net elknetwork \
  docker.elastic.co/beats/filebeat:7.16.3 filebeat -e -strict.perms=false \
  -E output.elasticsearch.hosts=["elasticsearch:9200"]

###########KIBANA CONFIGURATION#########################################

docker run -d -v "$PWD/kibana.yml":/usr/share/kibana/config/kibana.yml --name kibana --net elknetwork -p 5601:5601 kibana:7.16.3

####################ELASTICSEARCH CONFIGURATION####################################
  #start elasticsearch docker containter
  docker run -d -v "$PWD/elasticsearch.yml":/usr/share/elasticsearch/config/elasticsearch.yml --name elasticsearch --net elknetwork -p 9200:9200  -e "discovery.type=single-node" elasticsearch:7.16.3

  #####Security in elaticsearch#######################
  #Exec into a running elasticsearch  docker container
  docker exec -it -u elasticsearch elasticsearch bash

  #Generate certs and download into config folder
  ./bin/elasticsearch-certutil cert -out config/elk.p12
  chown elasticsearch:root config/elk.p12

  ./bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
  ./bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password


  #Add to elastic.yml file
  xpack.security.enabled: true
  xpack.security.transport.ssl.enabled: true
  xpack.security.transport.ssl.verification_mode: certificate
  xpack.security.transport.ssl.keystore.path:: elk.p12
  xpack.security.transport.ssl.truststore.path:: elk.p12

  #Set Password
  bin/elasticsearch-setup-passwords interactive
  elastic_123
  #test access
  curl -u elastic:elastic_123 elasticsearch:9200/_cat/health

  ####REFERENCES##################
  https://www.elastic.co/guide/en/elasticsearch/reference/6.6/configuring-tls.html
  https://www.elastic.co/guide/en/elasticsearch/reference/6.6/configuring-security.html
