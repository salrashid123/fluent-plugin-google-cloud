#### Fluentd



- Install and Test Local

1)

    - Create a service_account JSON key on a GCP project
    - Assign IAM role _Logging Writer_.

    - cd fluent-plugin-google-cloud && mkdir certs```

    - Download JSON cert to `certs/application_default_credentials.json` as shown below

    Copy `fluent-plugin-google-cloud/lib/fluent/pugin/out_google_cloud.rb` to `certs/` folder as well:
    - ```cp lib/fluent/plugin/out_google_cloud.rb `pwd`/certs```

2) on laptop
```
    docker run -ti  -p 8888:8888 -v `pwd`/certs/:/etc/google/auth/ debian /bin/bash
```
3) install fluentd in container

    - Install `fluentd`
```    
    apt-get update
    apt-get install -y curl sudo gnupg make gcc vim procps

    curl -L https://toolbelt.treasuredata.com/sh/install-debian-stretch-td-agent3.sh | sh
```

4) Install fluent-plugin-google-cloud in container
```
    /opt/td-agent/embedded/bin/gem install fluent-plugin-google-cloud
```
5) Add Certificate and change permisssions
```
    chown td-agent:td-agent /etc/google/auth/application_default_credentials.json
    chmod go-rwx /etc/google/auth/application_default_credentials.json

    $ ls -lart /etc/google/auth/application_default_credentials.json
    -rw-r----- 1 td-agent td-agent 2332 Jan 16 20:52 /etc/google/auth/application_default_credentials.json
```

6)  Copy `out_google_cloud.rb` such that its read by `td-agent`
```
    cp /etc/google/auth/out_google_cloud.rb /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluent-plugin-google-cloud-0.7.5/lib/fluent/plugin/out_google_cloud.rb 
```

7) Edit `/etc/td-agent/td-agent.conf`

Add

```
<filter gcp_resource.**>
  @type record_transformer
  @log_level debug
  <record>
    "logging.googleapis.com/local_resource_id" "generic_node.us-central1-a.default.somehost"
  </record>
</filter>
<match gcp_resource.**>
  @type google_cloud
  use_metadata_service false
  @log_level debug
</match>

<filter gcp_task.**>
  @type record_transformer
  @log_level debug
  <record>
    "logging.googleapis.com/local_resource_id" "generic_task.us-central-1.default.randomjob.randomtaskid"
  </record>
</filter>
<match gcp_task.**>
  @type google_cloud
  use_metadata_service false  
  @log_level debug
</match>



```

8) restart fluentd

   ``` service td-agent restart```

9) Send sample traffic to http listener
```
    curl -X POST -d 'json={"foo":"bar"}' http://localhost:8888/gcp_resource.log
    curl -X POST -d 'json={"foo":"bar"}' http://localhost:8888/gcp_task.log
```
10) tail log file

 ```   tail -f /var/log/td-agent/td-agent.log```



## TEST CASES
```
bundle exec ruby test/plugin/test_out_google_cloud.rb -n test_generic_node_default
bundle exec ruby test/plugin/test_out_google_cloud.rb -n test_generic_node_gce
bundle exec ruby test/plugin/test_out_google_cloud.rb -n test_generic_node_ec2

bundle exec ruby test/plugin/test_out_google_cloud.rb -n test_generic_task_default
bundle exec ruby test/plugin/test_out_google_cloud.rb -n test_generic_task_gce
bundle exec ruby test/plugin/test_out_google_cloud.rb -n test_generic_task_ec2
```