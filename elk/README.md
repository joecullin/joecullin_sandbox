# Trying ELK

Notes from trying logstash.
(Barely touched elastic or kibana so far.)

Table of contents:
* Logstash
* Elastic
* Environment setup: see _Docker container setup & notes_ at the bottom of this readme.


---
## Logstash

### Sample project overview:

Take a one-line input like `2020-02-19 13:24:05 hello joe`, and parse it into these fields:
* yyyy-mm-dd
* hour
* minute
* second
* message

Config file to edit:
```
joecullin_sandbox/elk/docker-elk/logstash/pipeline/logstash.conf
```

Restart logstash:
```
docker-compose stop logstash && docker-compose start logstash && docker-compose logs -f --tail=10 logstash
```

Test:
```
# valid input:
echo "2020-02-19 13:24:05 hello joe" | nc -v localhost 5000

# invalid input:
echo "2020-02-19     13:24:05 hello joe" | nc -v localhost 5000
```




### parse input

Reference:
- *grok* basics: https://www.elastic.co/guide/en/logstash/7.6/plugins-filters-grok.html
- grok regexes: https://github.com/kkos/oniguruma/blob/master/doc/RE
- grok debugger: http://grokdebug.herokuapp.com/
- grok patterns: https://github.com/elastic/logstash/blob/v1.4.2/patterns/grok-patterns
  - TIMESTAMP_ISO8601 allows for space instead of 'T'
- nested fields: use brackets. (one-level fields can omit brackets) https://www.elastic.co/guide/en/logstash/current/event-dependent-configuration.html#logstash-config-field-references
- Use `[@metadata][xxx]` for temp xxx fields that you don't want to include in output.
- *dissect* is faster and simpler for splitting simple delimited data: https://www.elastic.co/guide/en/logstash/7.6/plugins-filters-dissect.html



### mutate

https://www.elastic.co/guide/en/logstash/current/plugins-filters-mutate.html

Put it all together into a nice one-line summary:
```
    mutate {
        add_field => {
            "summary" => "On %{date} (at hour=%{hour}, minute=%{minute}, second=%{second}) we got this message: '%{message_text}'"
        }
    }
```


### date filter

https://www.elastic.co/guide/en/logstash/current/plugins-filters-date.html


### json output
trying json codec for output, but it's not showing anything.
Confirmed the plugin is already installed:
https://www.elastic.co/guide/en/logstash/current/working-with-plugins.html
```
cd /usr/share/logstash/
logstash-plugin list
```
... hey, it's there now! Must've had something mis-typed before? Or maybe adding the ids helped? Or maybe I just skimmed over it as noise, since it's not pretty-printed? ... actually I just noticed the json output lags one message behind. Is it b/c it's missing a newline?

### how to install new logstash plugin

The LS_JAVA_OPTS setting is because I got this: `Error: Your application used more memory than the safety cap of 256M.`

```
docker exec -u 0 -it 7769f996146d bash

cd /usr/share/logstash
LS_JAVA_OPTS="-Xms256m -Xmx512m" bin/logstash-plugin install logstash-input-rss
```

### how to reload config (faster than stopping/starting whole container)
- logstash is always pid 1 in this docker container.
- See also https://www.elastic.co/guide/en/logstash/current/reloading-config.html for a setting to auto-reload config at a specified interval.
- Bigger non-config changes (like installing a new plugin) may require restart.
```
docker exec -u 0 -it 7769f996146d bash
kill -HUP 1
```

Some changes require full restart.
Useful for rapid edit/test cycles in that case:
```
docker-compose start logstash && docker-compose logs -f --tail=10 logstash ; docker-compose stop logstash
```


### Example output using elasticsearch plugin

- The hostname within this docker network is elasticsearch.
- SIGHUP doesn't re-connect. Stop & start the logstash container if you change the connection info.

```
    elasticsearch {
        hosts => ["http://elasticsearch:9200"]
        index => "%{index_name}"
        document_id => "%{request_id}"
        user=> "joetest"
        password => "joetest"
    }
```


---
## Elastic

Insert a record:

```
curl -uelastic:changeme -v 'http://localhost:9200/test1/_doc/curl' -X PUT -H 'Content-type: application/json' -d '{"prop1": "test"}'
```

Simple search:

```
curl -uelastic:changeme -v 'http://localhost:9200/test1/_search'
```

---
## Docker container setup & notes

Using this premade config:
https://logz.io/blog/elk-stack-on-docker/
https://github.com/deviantony/docker-elk

`git clone https://github.com/deviantony/docker-elk.git`

Init:
```
pushd /Users/Joe.Cullin/code/elk_sandbox/docker-elk && docker-compose up -d && popd
```
Cleanup: important to include the -v, to remove data volume?
```
docker-compose down -v
```

Start:
```
pushd /Users/Joe.Cullin/code/elk_sandbox/docker-elk && docker-compose start && popd
```
Stop:
```
pushd /Users/Joe.Cullin/code/elk_sandbox/docker-elk && docker-compose stop && popd
```

Restart logstash to try new config:
```
docker stop 7769f996146d && docker start 7769f996146d && docker logs -f 7769f996146d
```

__Skipped this for now:__ (it's just a throwaway sandbox on my machine.)
> The stack is pre-configured with the following privileged bootstrap > user:
> * user: elastic
> * password: changeme
> 
> Although all stack components work out-of-the-box with this user, we strongly recommend using the unprivileged built-in users instead for increased security.
...
...


Kibana works: http://localhost:5601/

Send content to Logstash via TCP:

```
echo "Joe test 1" | nc -v localhost 5000
echo "2020-02-19 13:24:05 hello joe" | nc -v localhost 5000
```

No impact though.
> Check out logstash container... `docker exec -it 9b1468adac1f /bin/bash` ... abandoned that tangent ... did the next step. Turned out the log entries were there all along.

Set up an index in Kibana:
> Navigate to the Discover view of Kibana from the left sidebar. You will be prompted to create an index pattern. Enter logstash-* to match Logstash indices then, on the next page, select @timestamp as the time filter field. Finally, click Create index pattern and return to the Discover view to inspect your log entries.

Exploring logstash conf

https://www.elastic.co/guide/en/logstash/current/configuration.html

~/code/elk_sandbox/docker-elk/docker-stack.yml has a pointer to the config files:
    configs:
      - source: logstash_config
        target: /usr/share/logstash/config/logstash.yml
      - source: logstash_pipeline
        target: /usr/share/logstash/pipeline/logstash.conf

Logstash config files on my mac:
mac master clean ~/code/elk_sandbox/docker-elk/logstash $ find . -type f
./pipeline/logstash.conf
./config/logstash.yml
./Dockerfile
