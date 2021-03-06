
## Logging solution for WebSphere Commerce on Kubernetes ##

 This topic demonstrates how to use ELK (Elastic) stack, a popular open-source logging platform, as a sample logging solution for WebSphere Commerce deployment utilities. In your production environment, you can also leverage other commercial logging solutions to meet your business needs.
 In this sample ELK solution, the following components are included:
 * **ElasticSearch**: Is an open-source, distributed, RESTful, JSON-based search engine.
 * **Kibana**:  Allows you to visualize your Elasticsearch data and navigate the ELK Stack.
 * **Filebeat**: Offers a lightweight way to forward and centralize logs and files.

For more information about the ELK stack, see [the Elastic website](https://www.elastic.co/elk-stack).

**Note**: This sample solution leverages the following configuration files provided in the project under the `utilities/filebeat/` directory.
:
* `Entrypoint.sh`
* `filebeat.yml`
* `dockerfile`

The following diagram shows the architecture of the sample solution. Each WebSphere Commerce Docker container works with a 'sidecar' Filebeat container to collect logs in real time. By leveraging the shared volume capability of pod, Filebeat can access the log files in the shared volume of WebSphere Commerce directly, and then transfer the logs (in JSON format) to the ElasticSearch system. After that you can view the logs in the Kibana UI. <br/>
<img src="./images/WC_K8S_ELK.png" width = "800" height = "450" align=center />

After Kubernetes launches all the containers, you can check the logs in the Kinaba UI through `http://<Kibana_Host>:5601`, as shown below.<br/>
  <img src="./images/Kibana1.png" width = "800" height = "450" align=center /><br/>

  The message field in the Kibana UI includes a log message, as shown below.<br/>
<img src="./images/KibanaMessageField.png" width = "600" height = "335" align=center /><br/>
  <what we customize>

## Building customized Filebeat Docker image##

To simplify the deployment process in Kubernetes, in the sample solution, we put the customized Filebeat configuration file `filebeat.yml` and the Docker initialization file `Entrypoint.sh` in the customized Filebeat Docker image. In this way, you only need to build the customized image once, and it can apply to varies WebSphere Commerce container components, such as Search, Foundation, etc. by pointing to different `filebeat.yml` in the `Entrypoint.sh` script. You can find the sample file in the project under the `utilities/filebeat/` directory.


To do so, pull the Filebeat Docker image from the official repository (you'd better have a local repository), modify the sample dockerfile, `Entryponit.sh` and `Filebeat.yml` based on your needs, and build the customized Filebeat image. <br/>

Sample Dockerfile
<pre>
FROM filebeat:6.3.0
COPY Entrypoint.sh /etc/
COPY filebeat_search_store.yml /etc/
COPY filebeat_ts_app.yml /etc/
COPY filebeat_ts_web.yml /etc/
USER root
ENTRYPOINT ["/etc/Entrypoint.sh"]
</pre>

Build customized filebeat image
<pre>
docker build -f &lt;sample_dockerfile&gt; -t &lt;repository&gt;:&lt;tag&gt; .
</pre>

<!--
<img src="https://github.com/IBM/wc-devops-utilities/raw/draftdoc/doc/images/Build_Filebeat_Image.png" width = "600" height = "135" align=center /><br/>
-->
  ### Customizing `filebeat.yml` ###

  To get rid of the exception stack split into multiple messages in the ElasticSearch, add the following to the `filebeat.yml` file:
  <pre>
  multiline.pattern: '^\['
  multiline.negate: true
  multiline.match: after
  </pre>

  To add a new field to identify and filter logs by container types, add the following to the `filebeat.yml` file:
  <pre>
    fields:
      containerType: search
  </pre>
  <img src="./images/ContainerType.png" width = "300" height = "200" align=center /><br/>

  For more information about the `filebeat.yml` configuration, refer to:<br/>
  https://www.elastic.co/guide/en/beats/filebeat/current/configuring-howto-filebeat.html.

  ### Customizing `Entrypoint.sh` ###

  `Entrypoint.sh`gets the input of ‘-indexName' and  '-targetELK' parameters from Kubernetes, updates the `filebeat.yml` configuration file accordingly, and then uses the container specific `filebeat.yml` files (for search, tx, etc.) to launch the Filebeat service.<br/>
  For more details, refer to the Entrypoint.sh script.<br/>

  ### Customizing the dockerfile ###

  For detailed instructions, see the Docker official guide:<br/>
  https://docs.docker.com/engine/reference/builder/#usage

## Enabling the logging solution during WebSphere Commerce deployment ##
To enable the logging solution when you deploy WebSphere Commerce V9, follow these steps.
### Prerequisite ###
Before you enable the logging solution, you need to set up ElasticSearch and Kibana.
*  To install the ElasticSearch (or ElasticSearch cluster), follow the official installation guide Based on your business requirement, consider using the ElasticSearch cluster if the logging throughput is huge:<br/>
  https://www.elastic.co/guide/en/elasticsearch/reference/current/_installation.html<br/>
  https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html
* To install Kibana, follow the official installation guide:<br/>
  https://www.elastic.co/guide/en/kibana/current/install.html

 ### Procedure ###

 To enable the logging solution, launch the Filebeat container alone with the WebSphere Commerce container.

 You need to change the following flags in the parameters of the DeployWCSCloud_Base job. For details of the DeployWCSCloud_Base job, see [Using WebSphere Commerce Deployment Utilities](./DeployControllerDesign.md).
 * <B>FileBeat.Enable</B>: Change this flag to 'true' to enable Filebeat container alone with the WebSphere Commerce container.
 * <B>FileBeat.ELKServer</B>: Change this flag to your ElasticSearch server (or the the Master node of your ElasticSearch cluster).


 The following screen shot shows a sample deployment job.<br/><br/>
 <img src="./images/Enable_Filebat&ELK.png" width = "450" height = "400" align=center /><br/>


  ### Reference: The out-of-the-box behavior of the sample logging solution ###

  <img src="./images/Enable_Logging_logic.png" width = "500" height = "250" align=center /><br/>

  Using the flag 'FileBeat.Enable' in the Jenkins HelmChart_values field to contral enable the filebeat container alone with commerce container or not.<br/>


You can launch the Filebeat container alone with the WebSphere Commerce container using the build-in `filebeat.yml` file, which is built-in with the customized Filebeat image, or using a customized `filebeat.yml` file. If a customized `filebeat.yml` file is specfied, Kubernets will use the customized file  and ingore the built-in file.

You can specify the customized `filebeat.yml` file by creating Kubernetes configmaps. For example, define a specific Filebeat configuration file for the store container. Here are the steps:<br/>
1. Define the customized Filebeat configuration file (`filebeat-cus.yml`).
2. Create a configmap based on the customized Filebeat configuration file.
  <pre>
  kubectl create configmap filebeat-config-crs-app --from-file filebeat-cus.yml
  </pre>
3. Update the configmap in `values.yaml`. Set the value 'Values.Crsapp.FileBeatConfigMap' to 'filebeat-config-crs-app'.

The configmap will mount to the `/etc/filebeat/` folder of the Filebeat container. The container will start with the configuration file `/etc/filebeat/filebeat-cus.yml`.
