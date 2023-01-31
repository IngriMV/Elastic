# Envió de datos a través de Querys de Postgres hacia ELK

La siguiente guía contiene el paso a paso de envío de querys y configuración del '@Timestamp' para evitar duplicidad de data en el ELK.

#### Requerimientos

- Descargar [agente Beat logstash](https://www.elastic.co/es/downloads/past-releases#elastic-agent) De acuerdo a la versión del ELK
- - Instalar el agente **JDBC**, para este ejemploo se realizará en un Sistema Operativo Linux.

#### Paso 1. Instalación de agente Beat Logstash
Se descarga el agente  
```bash
curl -L -O  https://artifacts.elastic.co/downloads/logstash/logstash-8.6.1-amd64.deb
```
Se procede a instalar el agente

```bash
sudo dpkg -i logstash-8.4.2-amd64.deb
```


#### Paso 2. Instalación de JDBC 

Instalar la dependecia para este servicio, es necesario Java

```bash
sudo curl https://jdbc.postgresql.org/download/postgresql-{version}.jar -o /usr/share/logstash/logstash-core/lib/jars/postgresql-jdbc.jar
```

Se instala el plugin de JDBC 
```bash
sudo bin/plugin install logstash-input-jdbc
```

#### Paso 3. Configuración del archivo .conf

Se requiere la siguiente información por parte del administrador de Base de datos, el Usuario y contraseña de lectura, la URL de donde se aloja la Base de datos de donde se traerá la consulta.

```bash
jdbc_connection_string => "jdbc:postgresql://localhost:5432/{pg_database}"
jdbc_user => "{pg_username}"
jdbc_password => "{pg_password}"
```

De acuerdo a la documentación de Elastic [Diseño de directorios](https://www.elastic.co/guide/en/logstash/current/dir-layout.html), la ruta donde se crea los archivos con extensión `.conf` es:

```bash
cd /etc/logstash/conf.d/ 
sudo nano ejemplo.conf
```
Allí se crea el archivo:

```bash
# Sample Logstash configuration for creating a simple
# Beats -> Logstash -> Elasticsearch pipeline.

input {
  jdbc {
    jdbc_driver_library => "/usr/share/java/postgresql-42.2.10.jar"
    jdbc_driver_class => "org.postgresql.Driver"
    jdbc_connection_string => "jdbc:postgresql:////localhost:5432/{pg_database}"
    jdbc_user => ""
    jdbc_password => ""
    #schedule => "*/1 * * * *"
   #statement_filepath => "/etc/logstash/query.sql"
    statement => "select id::text AS user_id, ip_address::text AS postgresql_log_client_addr, time::timestamp AT TIME ZONE 'America/bogota' AS postgresql_log_session_start_time, boleanos::bool AS postgresql_activity_state, data::text AS message from * where time >= '2022¿3-01-30' and time >:sql_last_value order by updated_at desc;"
    type => "query1"
  }
}

filter {
   grok{
   match => {"message"=> ['.*"login": %{DATA:user.login},.*"error": \["%{DATA:error.code}"\].*',
                          '.*risk_score": %{NUMBER:event.risk_score}.*"time2": "%{TIMESTAMP_ISO8601:postgresql.activity.state_change}.*'
     ]

  }
         }

  mutate {

      rename => {
      "user_id" => "user.id"    
      "postgresql_log_client_addr" => "postgresql.log.client_addr"        
      "postgresql_activity_state"  => "postgresql.activity.state"
                }

        remove_field => ["message"]
        remove_tag => ["_grokparsefailure"]	
        remove_tag => ["_dateparsefailure"]
        remove_tag => ["[]"]       
    
  convert => {
          "event.risk_score" => "float"          
          "user.login" => "boolean"
          
        }  
  }

   date {
        match => ["postgresql.activity.state_change", "yyyy/MM/dd HH:mm:ss Z", "ISO8601"]
        target => ["postgresql.activity.state_change"]
      }
 

}
 
output {
    elasticsearch {
    hosts => ["hxxps://localhost.com:9243/"]
    user => ""
    password => ""
    data_stream => "true"
    data_stream_type => "logs"
    data_stream_dataset => "postgres.query1"
    data_stream_namespace => "default"       
   
  }
}

```

