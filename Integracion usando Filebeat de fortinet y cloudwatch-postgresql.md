# Integracion usando el beat (Filebeat) utilizando los modulos de fortinet y cloudwatch-postgresql

La sigiuente guía, es el paso a paso de la configuración de **filebeat.yml** para diferentes tipos de datos, para este ejemplo datos de fortinet y Cloudwatch.

# Paso 1. Configurar los inputs

Configurar las entradas del fortinet y Cloudwatch, en la parte del archivo **filebeat.yml**, en la sección de Inputs, se debe ver la lista de modulos, hay que activar para este ejemplo el modulo de fortinet de filebeat de la siguiente forma:

```
filebeat modules list
filebeat modules enable fortinet
```

Y se configura las dos diferentes entradas así:

```
# ============================== Filebeat inputs ===============================

filebeat.inputs:
- type: http_endpoint
  enabled: true
  listen_address: 192.168.1.1
  listen_port: 8080
  response_code: 200
  response_body: '{"message": "success"}'
  url: "/"
  prefix: "json"
  tags: ["cloudguard"]

- type: aws-cloudwatch
  enabled: true
  access_key_id: '${AWS_ACCESS_KEY_ID}'
  secret_access_key: '${AWS_SECRET_ACCESS_KEY}'
  log_group_arn: arn:aws:logs:us-east-1:428152502467:log-group:test:*
  start_position: end
  fields:
    source: aws_cloudwatch

# ============================== Filebeat modules ==============================

filebeat.config.modules:
  # Glob pattern for configuration loading
  path: ${path.config}/modules.d/*.yml

  # Set to true to enable config reloading
  reload.enabled: false

  # Period on which files under path should be checked for changes
  #reload.period: 10s
``` 
  
# Paso 2. configurar los templates e index

En la sección de template como son dos entradas de indices es necesario configurar la siguiente estructura:

```
# ======================= Elasticsearch template setting =======================

setup.template.settings:
  index.number_of_shards: 1
setup.ilm.enabled: false
output.elasticsearch:
  indices:
    - index: "logs-awspostgressql-default"
      when.contains:
        fields.source: "aws_cloudwatch"
      setup.template:
        name: "logs-awspostgressql-default"
        pattern: "logs-awspostgressql-default"

    - index: "logs-fortivpn-default"
      when.equals:
        event.module: "fortinet"
      setup.template:
        name: "logs-fortivpn-default"
        pattern: "logs-fortivpn-default"
``` 
 
> `when.equals`  se utiliza para comparar si el valor de un campo es igual a un valor específico, mientras que `when.contains` se utiliza para buscar si una cadena de texto específica se encuentra en el valor del campo indicado.
En resumen, la condición `when.equals` se utiliza para filtrar eventos según el valor exacto de un campo determinado (como en el ejemplo de event.module: "fortinet"), mientras que la condición `when.contains` se utiliza para filtrar eventos según si una cadena de texto específica se encuentra en el valor del campo (como en el ejemplo de fields.source: "aws_cloudwatch"). Estas condiciones se pueden utilizar en Filebeat para filtrar eventos de acuerdo a diferentes criterios.


# Paso 3. Configuración del output 
Como en este ejemplo se enviara los logs a una instancia cloud de elasticSearch, se configura así, en la sección de elastic Cloud:

```
# =============================== Elastic Cloud ================================
cloud.id: "staging:dXMtZWFzdC0xLmF3cy5mb3VuZC5pbyRjZWM2ZjI2MWE3NGJmMjRjZTMzYmI4ODExYjg0Mjk0ZiRjNmMyY2E2ZDA0MjI0OWFmMGNjN2Q3YTllOTYyNTc0Mw=="
cloud.auth: "elastic:YOUR_PASSWORD"
``` 

# Paso 4. configuración del “Grok” del filebeat

Para la configuración de los grok, en **filebeat.yml**, se utiliza la sección de los processors estos pueden realizar diferentes acciones, como agregar, eliminar o modificar campos, renombrar eventos, transformar el contenido del evento de log y mucho más. Los processors son muy útiles para realizar tareas de limpieza y normalización de los eventos de log, antes de ser almacenados o enviados a otros sistemas. Estos tienen la función Dissect, cuando se tiene un formato estructurado y consistente en los logs y se desea extraer información específica de los campos, mientras que se utiliza la función Grok cuando se tiene un formato menos estructurado y se desea identificar patrones complejos en los logs. Para más información [Processors](https://www.elastic.co/guide/en/beats/filebeat/current/filtering-and-enhancing-data.html)

```
# ================================= Processors =================================
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~

  - if:
      equals:
        input.type: aws-cloudwatch
    then:
      - dissect:
          tokenizer: "%{awspostgresql.log.timestamp} UTC:%{awspostgresql.log.client_addr}(%{awspostgresql.log.core_id}):%{awspostgresql.log.user}@%{awspostgresql.log.database}:[%{awspostgresql.log.session_id}]:%{awspostgresql.log.level}:%{},WRITE,%{awspostgresql.log.command_tag},,,%{awspostgresql.log.query_name}"
          field: "message"
          target_prefix: ""

      - dissect:
          tokenizer: "%{awspostgresql.log.timestamp} UTC:%{awspostgresql.log.client_addr}(%{awspostgresql.log.core_id}):%{awspostgresql.log.user}@%{awspostgresql.log.database}:[%{awspostgresql.log.session_id}]:%{awspostgresql.log.level}:%{},DDL,%{awspostgresql.log.command_tag},,,%{awspostgresql.log.query_name}"
          field: "message"
          target_prefix: ""

      - dissect:
          tokenizer: "%{awspostgresql.log.timestamp} UTC:%{awspostgresql.log.client_addr}(%{awspostgresql.log.core_id}):%{awspostgresql.log.user}@%{awspostgresql.log.database}:[%{awspostgresql.log.session_id}]:%{awspostgresql.log.level}:  %{awspostgresql.log.detail}"
          field: "message"
          target_prefix: ""

      - dissect:
          tokenizer: "%{awspostgresql.log.timestamp} UTC:%{awspostgresql.log.client_addr}(%{awspostgresql.log.core_id}):%{awspostgresql.log.user}@%{awspostgresql.log.database}:[%{awspostgresql.log.session_id}]:%{awspostgresql.log.level}:  %{awspostgresql.log.query_name}"
          field: "message"
          target_prefix: ""

      - timestamp:
          field: event.ingested
          target_field: "@timestamp"
          layouts:
            - '2006-01-02T15:04:05Z'
            - '2006-01-02T15:04:05.999Z'
            - '2006-01-02T15:04:05.999-07:00'
          test:
            - '2019-06-22T16:33:51Z'
            - '2019-11-18T04:59:51.123Z'
            - '2020-08-03T07:10:20.123456+02:00'

  - drop_fields:
      fields: ["host","mac","hostname","architecture","log.flags","agent.name"]
      ignore_missing: true
``` 
      
## Se ejecuta el servicio de filebeat:

**Linux:**
```
sudo service filebeat start
sudo systemctl enable filebeat
journalctl -u filebeat.service
```

**Windows:**
```
PS C:\Program Files\filebeat> Start-Service filebeat
```

## REFERENCIAS

* [Input http_endpoint](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-http_endpoint.html)
* [Input AWSCloudwatch](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-aws-cloudwatch.html)
* [Filebeat commandline](https://www.elastic.co/guide/en/beats/filebeat/current/command-line-options.html#modules-command)
* [Filtering filebeat](https://www.elastic.co/guide/en/beats/filebeat/current/filtering-and-enhancing-data.html)
* [Running service filebeat](https://www.elastic.co/guide/en/beats/filebeat/current/running-with-systemd.html)
