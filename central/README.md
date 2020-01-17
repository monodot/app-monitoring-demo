# central

Components to set up monitoring with Prometheus and Grafana in a central datacentre.

## To deploy

First update the file `.applier/inventory/group_vars/seed-hosts.yml` to set the correct parameters for your environment:

- full spec of the images for Prometheus, Grafana and OpenShift OAuth Proxy

- basic credentials for Prometheus

Ensure Ansible is installed in your environment and then run:

    ansible-galaxy install -r .applier/requirements.yml -p roles
    ansible-playbook .applier/apply.yml -i .applier/inventory/hosts -e filter_tags=prometheus -e namespace=YOUR_NAMESPACE
    ansible-playbook .applier/apply.yml -i .applier/inventory/hosts -e filter_tags=grafana -e namespace=YOUR_NAMESPACE

## To test

Now deploy an AMQ broker with enhanced metrics configuration:

    export MY_PROJECT=yourprojectname

    oc replace --force -n ${MY_PROJECT} -f \
    https://raw.githubusercontent.com/jboss-container-images/jboss-amq-7-broker-openshift-image/72-1.2.GA/amq-broker-7-image-streams.yaml
    oc import-image amq-broker-72-openshift:1.2 -n ${MY_PROJECT}
    oc process -f https://raw.githubusercontent.com/jboss-container-images/jboss-amq-7-broker-openshift-image/72-1.2.GA/templates/amq-broker-72-basic.yaml \
        -p IMAGE_STREAM_NAMESPACE=${MY_PROJECT} \
        -p AMQ_USER=admin -p AMQ_PASSWORD=admin | oc apply -n ${MY_PROJECT} -f -

    # Update the broker with enhanced JMX metrics, courtesy of Laurent Broudoux!

    oc new-build ${MY_PROJECT}/amq-broker-72-openshift:1.2~https://github.com/lbroudoux/openshift-cases --name=custom-amq-broker-72-openshift --context-dir=jboss-amq7-custom/custom-amq -n ${MY_PROJECT}
    oc set triggers dc/broker-amq --containers=broker-amq --from-image=custom-amq-broker-72-openshift:latest -n ${MY_PROJECT}
    oc set env dc/broker-amq JAVA_OPTS="-Dcom.sun.management.jmxremote=true -Djava.rmi.server.hostname=127.0.0.1 -Dcom.sun.management.jmxremote.port=1099 -Dcom.sun.management.jmxremote.ssl=true -Dcom.sun.management.jmxremote.registry.ssl=true -Dcom.sun.management.jmxremote.ssl.need.client.auth=true -Dcom.sun.management.jmxremote.authenticate=false -javaagent:/opt/amq/lib/optional/jmx_prometheus_javaagent-0.11.0.jar=9779:/opt/amq/conf/prometheus-config.yml" -n ${MY_PROJECT}

    oc patch dc/broker-amq --type=json -p '[{"op":"add", "path":"/spec/template/spec/containers/0/ports/-", "value": {"containerPort": 9779, "name": "prometheus", "protocol": "TCP"}}]' -n ${MY_PROJECT}
    oc expose dc/broker-amq --name=broker-amq-prometheus --port=9779 --target-port=9779 --protocol="TCP" -n ${MY_PROJECT}
    oc annotate svc/broker-amq-prometheus prometheus.io/scrape='true' prometheus.io/port='9779' prometheus.io/path='/' -n ${MY_PROJECT}


TODO:

- Fix issue with unable to authenticate using OAuth - "The request is missing a required parameter, includes an invalid parameter value, includes a parameter more than once, or is otherwise malformed."
