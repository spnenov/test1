## Flink DataDog integration

The Flink Datadog integration enables streamlined monitoring, metrics tracking, and log management across Kubernetes resources.\
In order to achieve this integration, the following steps must be carried out.

Step 1 Set **Worker name**.\
The name in the Chart.yaml file `flink-pipeline-template` must be meaningful, as it will be used in all files in `chart/` folder and will represent the **worker's name**.
For example, in the following repository `dataeng-dip-etl-dt-auth-acquirerrequested`, the name in the Chart.yaml is represented as follows: `flink-dip-acquirer-requested`.

If you want to use this template for another pipeline you have to change all references of `flink-pipeline-template` in all files in `chart/` folder to `your_new_pipeline_name` for example to `flink-dip-acquirer-requested`.
The fastest way doing this is using the command 
```grep -rl flink-pipeline-template chart/ | xargs sed -i 's/flink-pipeline-template/flink-dip-acquirer-requested/'```

Step 2 Introduce the secret and all the related components around it.\
Step 2.1 Create **externalsecret-datadogapi.yaml**.\
The **externalsecret-datadogapi.yaml** file sets up an ExternalSecret in Kubernetes to securely retrieve the Datadog API key from an external secret manager when datadogConfig.enabled is true. 
The name `externalsecret-datadog-{{ include "flink-pipeline-template.fullname" . }}` gives a unique identifier for the ExternalSecret, while `{{ include "flink-pipeline-template.labels" . | nindent 4 }}` applies standardized labels.
The spec section specifies the refresh interval for fetching the secret, the reference to the secret store, and the key-value mapping for the API key, ensuring the secret is kept up-to-date and accessible to the application.

Example of the **externalsecret-datadogapi.yaml** file:
```
{{- if .Values.datadogConfig.enabled }}
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: externalsecret-datadog-{{ include "flink-pipeline-template.fullname" . }}
  labels:
    {{- include "flink-pipeline-template.labels" . | nindent 4 }}
spec:
  refreshInterval: 12h
  secretStoreRef:
    kind: ExternalSecret
    name: {{ include "flink-pipeline-template.fullname" . }}-secrets
  data:
  - secretKey: apiKey
    remoteRef:
      key: {{ .Values.datadogConfig.smSecret }}
      property: api-key
      conversionStrategy: Default
      decodingStrategy: None
      metadataPolicy: None
{{- end }}
```
Step 2.2 Create **secretstore.yaml**.\
The **secretstore.yaml** file defines a SecretStore resource in Kubernetes to securely connect to AWS Secrets Manager when datadogConfig.enabled is true.
It uses JWT authentication with a specified service account to securely retrieve secrets from the corresponding AWS region.

Example of the **secretstore.yaml** file:
```
{{- if .Values.datadogConfig.enabled }}
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: {{ include "flink-pipeline-template.fullname" . }}-secrets
  labels:
    {{- include "flink-pipeline-template.labels" . | nindent 4 }}
spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-west-1
      auth:
        jwt:
          serviceAccountRef:
            name: eso-flink-sa-{{ include "flink-pipeline-template.serviceAccountName" . }}
{{- end }}
```
Step 2.3 Create **serviceaccount-secretstore.yaml**.\
The **serviceaccount-secretstore.yaml** file creates a Kubernetes ServiceAccount named `eso-flink-sa-{{ include "flink-pipeline-template.serviceAccountName" . }}` when datadogConfig.enabled is true,
with an optional annotation for an AWS role ARN if smSecretRole is provided, enabling secure access on the ServiceAccount to AWS Secrets Manager for fetching secrets.

Example of the **serviceaccount-secretstore.yaml** file:
```
{{- if .Values.datadogConfig.enabled }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eso-flink-sa-{{ include "flink-pipeline-template.serviceAccountName" . }}
  labels:
    {{- include "flink-pipeline-template.labels" . | nindent 4 }}
  {{- if .Values.datadogConfig.smSecretRole }}
  annotations:
    eks.amazonaws.com/role-arn: {{.Values.datadogConfig.smSecretRole }}
  {{- end }}
{{- end }}
```
Step 3 Confugiration on **flinkdeployment.yaml**.\
The following configuration on **flinkdeployment.yaml** enables Datadog integration for monitoring and logging in Flink.
It sets the DD_APIKEY environment variable to securely fetch the Datadog API key from the Kubernetes ExternalSecret named `externalsecret-datadog-{{ include "flink-pipeline-template.fullname" . }}`, 
using the apiKey reference. Then, if Datadog is enabled and `flinkConfDir` is defined, it sets the `FLINK_CONF_DIR` environment variable accordingly. 
Additionally, it enables Datadog metrics and log labeling, applying custom log labels to JobManager and TaskManager pods, designating Datadog as the HTTP metrics reporter in the EU data center, 
enabling logical identifiers, defining Flink-specific metrics scopes, and adding extra metrics tags through dd-labels-metrics. It sets up a log4j-console.properties configuration that logs to the console in a compact JSON format, with log levels set for common libraries and suppresses certain Netty warnings to ensure clean and informative output.

Example from **flinkdeployment.yaml**:


#dd_apikey
```
            - name: DD_APIKEY
              valueFrom:
                secretKeyRef:
                  name: externalsecret-datadog-{{ include "flink-pipeline-template.fullname" . }}
                  key: apiKey
```
#Flink conf dir
```
            {{- if and (.Values.datadogConfig.enabled) (.Values.datadogConfig.flinkConfDir) }}
            - name: FLINK_CONF_DIR
              value: "{{ .Values.datadogConfig.flinkConfDir }}"
            {{- end}}
```
#Datadog metrics
```
    {{- if .Values.datadogConfig.enabled }}
    kubernetes.jobmanager.labels: {{ include "flink-pipeline-template.dd-labels-logs" . }}
    kubernetes.taskmanager.labels: {{ include "flink-pipeline-template.dd-labels-logs" . }}
    metrics.reporter.dghttp.factory.class: org.apache.flink.metrics.datadog.DatadogHttpReporterFactory
    metrics.reporter.dghttp.dataCenter: EU
    metrics.reporter.dghttp.useLogicalIdentifier: "true"
    metrics.scope.jm: flink.jobmanager
    metrics.scope.jm.job: flink.jobmanager.job
    metrics.scope.tm: flink.taskmanager
    metrics.scope.tm.job: flink.taskmanager.job
    metrics.scope.task: flink.task
    metrics.scope.operator: flink.operator
    metrics.reporter.dghttp.scope.variables.additional: {{ include "flink-pipeline-template.dd-labels-metrics" . }} 
    {{- end }}
```
#logConfiguration
```
  {{- if .Values.datadogConfig.enabled }}
  logConfiguration:
    "log4j-console.properties": |
      # This affects logging for both user code and Flink
      rootLogger.level = INFO
      rootLogger.appenderRef.console.ref = LogConsole

      # Log all to the console using compact JSON format
      appender.console.name = LogConsole
      appender.console.type = CONSOLE
      appender.console.layout.type = JsonLayout
      appender.console.layout.compact = true
      appender.console.layout.eventEol = true

      # The following lines keep the log level of common libraries/connectors on
      # log level INFO. The root logger does not override this. You have to manually
      # change the log levels here.
      logger.akka.name = akka
      logger.akka.level = INFO
      logger.kafka.name= org.apache.kafka
      logger.kafka.level = INFO
      logger.hadoop.name = org.apache.hadoop
      logger.hadoop.level = INFO
      logger.zookeeper.name = org.apache.zookeeper
      logger.zookeeper.level = INFO

      # Suppress the irrelevant (wrong) warnings from the Netty channel handler
      logger.netty.name = org.apache.flink.shaded.akka.org.jboss.netty.channel.DefaultChannelPipeline
      logger.netty.level = OFF
  {{- end }}
```

Step 4 `datadogConfig` in the **values** files.\
The `fullnameOverride` name in the **values** files for different environments (such as **Dev**, **Uat**, etc.) overrides the chart name specified in the **Chart.yaml** file.
The below example from **values-dev.yaml** file shows how the worded name should look.
`fullnameOverride: "dip-dt-auth-acquirer-requested"`

The below `datadogConfig` section in the velues files enables Datadog integration, setting the Flink conf path, AWS Secrets Manager Api key(smSecret) and role ARN(smSecret),
along with custom lablels to tag pods and metrics with environemnt specific metadata for monitoring.

Example from **values-dev.yaml** file:
```
datadogConfig:
  enabled: true     #indicates that the Datadog integration is active.
  flinkConfDir: /opt/etl/conf   #specifies the directory path where the Flink configuration files are located.
  smSecret: datadog-api-key     #the name of the secret containing the Datadog API key. 
  smSecretRole: arn:aws:iam::975050235153:role/EKS-eu-west-1-ESO-flink      #AWS role ARN for secret access.
  labels: # pods lables with added "tags.datadoghq.com/" also used as metrics tags
    env: dip-dev    #sets an environment label, which is applied to the pods.
    level1: TBC
    level2: TBC
    level3: TBC
    businessUnit: Planet-Portal    #specifies a label that categorizes the pods by the business unit.
```

Step 5 Confugiration on **_helpers.tpl**.\
The following DataDog functions were introduced in **_helpers.tpl**:\
Internal Labels Function - `_flink-pipeline-template.dd-labels`: Defines default Datadog labels by setting service to the fully qualified chart name (as defined by `flink-pipeline-template.fullname`)
and version to the image tag or "latest", then outputs them in JSON format for consistent use.\
Metrics Labels - `flink-pipeline-template.dd-labels-print-metrics`: Iterates over each label key-value pair, formatting them as key:value and separating each with a comma for metrics tagging.
`flink-pipeline-template.dd-labels-metrics`: Calls the dd-labels-print-metrics function to format the JSON labels for metrics, then removes any trailing comma.\
Logs Labels - `flink-pipeline-template.dd-labels-print-logs`: Formats each label for log tagging by adding the tags.datadoghq.com/ prefix to the key-value pairs, separated by commas.
`flink-pipeline-template.dd-labels-logs`: Calls the dd-labels-print-logs function, passing JSON-formatted labels, and trims any trailing commas to ensure a clean output for Datadog log tagging.

Example from the **_helpers.tpl** file:

```
{{/*
Internal labels function
*/}}
{{- define "_flink-pipeline-template.dd-labels" -}}
{{- $labels := default (dict) .Values.datadogConfig.labels }}
{{- if not (hasKey $labels "service") }}
{{- $_ := set $labels "service" (include "flink-pipeline-template.fullname" .) }}
{{- end }}
{{- if not (hasKey $labels "version")}}
{{- $_ := set $labels "version" (default "latest" .Values.image.tag) }}
{{- end }}
{{- mustToJson $labels  }}
{{- end }}

{{/*
Metrics labels
*/}}

{{- define "flink-pipeline-template.dd-labels-print-metrics" -}}
{{- range $key, $value := . }}{{( print $key ":" $value ", " ) }}{{ end }}
{{- end }}

{{- define "flink-pipeline-template.dd-labels-metrics" -}}
{{- include "flink-pipeline-template.dd-labels-print-metrics" (mustFromJson ( include "_flink-pipeline-template.dd-lables" . )) | trimSuffix ", "}}
{{- end }}

{{/*
Logs labels
*/}}

{{- define "flink-pipeline-template.dd-labels-print-logs" -}}
{{- range $key, $value := . }}{{( print "tags.datadoghq.com/" $key ":" $value ", " ) }}{{ end }}
{{- end }}

{{- define "flink-pipeline-template.dd-labels-logs" -}}
{{- include "flink-pipeline-template.dd-labels-print-logs" (mustFromJson (include "_flink-pipeline-template.dd-labels" .)) | trimSuffix ", "}}
{{- end }}
```

Step 6 Confugration on the **docker-entrypoint.sh** script.\
The **docker-entrypoint.sh** script facilitates the setup of the Flink environment by initializing configuration files and setting important options,
ensuring that the application is correctly configured before it starts running.The `FLINK_CONF_DIR` variable defaults to `/opt/etl/conf` if not explicitly defined, and it creates that directory along with an empty `flink-conf.yaml` file within it. 
The script then copies the default `flink-conf.yaml` from `/opt/flink/conf/` to the new configuration directory, setting the `metrics.reporter.dghttp.apikey` option to the value of `DD_APIKEY`. 
Finally, it transfers the `log4j-console.properties` file to the new directory, ensuring proper logging configurations are in place.

Example from **docker-entrypoint.sh**:
```
FLINK_CONF_DIR=${FLINK_CONF_DIR:-/opt/etl/conf}
CONF_FILE="${FLINK_CONF_DIR}/flink-conf.yaml"
mkdir -p $FLINK_CONF_DIR
touch $CONF_FILE
```
```
#We override the default config file location and add some config options here
cat /opt/flink/conf/flink-conf.yaml > ${FLINK_CONF_DIR}/flink-conf.yaml
set_config_option  metrics.reporter.dghttp.apikey ${DD_APIKEY}
cat /opt/flink/conf/log4j-console.properties >  ${FLINK_CONF_DIR}/log4j-console.properties
```

Step 7 Logging on the **Dockerfile**.
The follwoing three libraries are added to the **Dockerfile** as they are relate to having JSON logging instead of textual ones, which allows DataDog to group the logs in a more normal way.

The additional libraries in the **Dockerfile** are as follows:
```
ADD --chown=flink:flink https://repo.maven.apache.org/maven2/com/fasterxml/jackson/core/jackson-databind/2.18.0/jackson-databind-2.18.0.jar $FLINK_HOME/lib/
ADD --chown=flink:flink https://repo.maven.apache.org/maven2/com/fasterxml/jackson/core/jackson-core/2.18.0/jackson-core-2.18.0.jar $FLINK_HOME/lib/
ADD --chown=flink:flink https://repo.maven.apache.org/maven2/com/fasterxml/jackson/core/jackson-annotations/2.18.0/jackson-annotations-2.18.0.jar $FLINK_HOME/lib/
```
These libraries are related to the `logConfiguration` specified in the below example from **flinkdeployment.yaml**, where we have set all logs to be output to the console in a compact JSON format.

Example of the `LogConfiguration` in **flinkdeployment.yaml**:
```
      # Log all to the console using compact JSON format
      appender.console.name = LogConsole
      appender.console.type = CONSOLE
      appender.console.layout.type = JsonLayout
      appender.console.layout.compact = true
      appender.console.layout.eventEol = true
```
