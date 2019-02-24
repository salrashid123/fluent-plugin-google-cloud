#### GCE

1) Create VM with `Logging Writer` scope enabled

2) `ssh` to the VM and install the structured logging agent

```
    curl -sSO "https://dl.google.com/cloudagents/install-logging-agent.sh"
    sudo bash install-logging-agent.sh --structured
```
2b) copy fluent-plugin-google-cloud/lib/fluent/plugin/out_google_cloud.rb to the VM
   then
```
    cp out_google_cloud.rb /opt/google-fluentd/embedded/lib/ruby/gems/2.4.0/gems/fluent-plugin-google-cloud-0.7.5/lib/fluent/plugin/out_google_cloud.rb

```

3) Add configuration to `/etc/google-fluentd/google-fluentd.conf`
```
<source>
  @type http
  @id input_http
  port 8888
</source>
<filter gcp_resource.**>
  @type record_transformer
  @log_level debug
  <record>
    "logging.googleapis.com/local_resource_id" "generic_node.us-central1-a.default.somehost"
  </record>
</filter>
<match gcp_resource.**>
  @type google_cloud
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
  @log_level debug
</match>



```


4) Restart `google-fluentd`

   ```  service google-fluentd restart```

5) Send sample traffic to http listner

```
    curl -X POST -d 'json={"foo":"bar"}' http://localhost:8888/gcp_resource.log
    curl -X POST -d 'json={"foo":"bar"}' http://localhost:8888/gcp_task.log
```

## TEST CASES

```
bundle exec ruby test/plugin/test_out_google_cloud.rb -n test_generic_node_default
bundle exec ruby test/plugin/test_out_google_cloud.rb -n test_generic_node_gce
bundle exec ruby test/plugin/test_out_google_cloud.rb -n test_generic_node_ec2

bundle exec ruby test/plugin/test_out_google_cloud.rb -n test_generic_task_default
bundle exec ruby test/plugin/test_out_google_cloud.rb -n test_generic_task_gce
bundle exec ruby test/plugin/test_out_google_cloud.rb -n test_generic_task_ec2
```