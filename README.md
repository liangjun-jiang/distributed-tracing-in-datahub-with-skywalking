# Applicaiton Performance Monitor and Distrubted Tracing with Apache SkyWalking in Datahub 


## About Datahub
LinkedIn's open source project [Datahub](https://linkedin.github.io/datahub/) is a gneeralized metadata search & discovery tool. It has been gaining popularity lately. 
It has very nice architecture to support not only bring in metadata into this tool via event-driven approach, but also support metadata discovery and search in a restful way. 
![datahub architecture](https://github.com/linkedin/datahub/blob/master/docs/imgs/datahub-architecture.svg)
As a summary, to use Datahub, you will use the following services. One type of service is that developed services, which consists of 
1. Datahub Frontend
2. GMS (Generalized Metadata Service) - think this as the main backend service
3. MCE Consumer Job - a Kafka consumer job to consume messages of MetadataChangeEvent(MCE) topic 
4. MAE Consumer Job - a Kafka consumer job to consumer messages of MetadataAuditEvent(MAE) topic

The other type service is leveraged services 
1. such as a relational database (MySQL, MariaDB, postgreSQL) 
2. Graph Database (Neo4j) 
3. Search Database (Elastic Search) 
4. Message Broker (Kafka) 

To get Datahub up and running, you will at least get those 8 services working. 
> If you have not tried Datahub, don't be scared. Datahub does provide `[quickstart.sh](https://github.com/linkedin/datahub/blob/master/docker/quickstart.sh)` to make the process easier. 

While deploy them into a production environment, or at least for your trial & error experiment, you might want to understand a little bit more about the peformance of Datahub.  

This is where Application Performance Monitor(APM) tool comes into the picture. Here we chosed [Apache SkyWalking](https://skywalking.apache.org/). The mission statement of Apache Skywalking is 
> Apache SkyWalking - Application performance monitor tool for distributed systems, especially designed for microservices, cloud native and container-based (Docker, K8s, Mesos) architectures

Why not [OpenTracing](https://opentracing.io/)? [Jaeger](https://www.jaegertracing.io/) ? [zipkin](https://zipkin.io/)? 

There are two things that affected my decision while I research this topic, also combining my experience with distributed tracking while working for Alibaba.
1. this Medium Article: [Distributed Tracing — we’ve been doing it wrong](https://medium.com/@copyconstruct/distributed-tracing-weve-been-doing-it-wrong-39fc92a857df)
2. The creator of SkyWalking, [Shen Wu](https://github.com/wu-sheng)'s talk: [Apache Skywalking, with Sheng Wu](https://www.youtube.com/watch?v=5dnNVz45jrA), and [this in Chinese](https://www.bilibili.com/video/BV1qV41167tj)

There maybe  one reason for you to use Apache Skywalking - it doesn't change single piece of your project to get it working with your project.


## About Apache SkyWalking
Visting [SkyWalking's official website](https://skywalking.apache.org/) is the best way to know more about this project. 
As a summary, Skywalking works in this way
1. You deploy an Skywalking agent with your application
2. You run a Skwyalking backend (OAP) and UI to collect and analysis telemetry data Skywalking agent sends. You also need to have a ElasticSearch (ES) service to help Skywalking OAP. 

Datahub is developed with Java. 


## Set up for Local Environment
1. Set up LinkedIn's Datahub
Even though LinkedIn's `[quickstart.sh](https://github.com/linkedin/datahub/blob/master/docker/quickstart.sh)` will help you spin up each service listed above. But right now, we actually just want to start with MySQL, ES, Neo4j and Kafka. We will run `GMS`, `mce-consumer-job` and `mae-consumer-job` on our own. 




2. Set up Apache Skywalking OAP and UI
You might agree that the eaiser way to set up Apache Skywalking OAP and UI is using docker-compose. You can find the offical `SkyWalking-Docker` [here](https://github.com/apache/skywalking-docker)
It provides different combination of SkyWalking version and Elastic Search (ES) version. 
Here is an example of [Skywalking OAP 8.1.0 with ES 7](https://github.com/apache/skywalking-docker/blob/master/8/8.1.0/compose-es7/docker-compose.yml)
You might have realized, since both Datahub and Skywalking OAP are using ElasticSearch, it's better idea that you just want to spin up one Elastic Search.   
The simplest way to do so is that we remove Elastic Search (ES) service from SkyWalking's docker compose file, and make sure that the docker containers of Skywalking OAP and UI are the same network as Datahub. 
An sample modifide docker-compose.yml might look like
```
# See the License for the specific language governing permissions and
# limitations under the License.

version: '3.8'
services:
  # elasticsearch:
  #   image: docker.elastic.co/elasticsearch/elasticsearch:6.8.6
  #   container_name: elasticsearch
  #   restart: always
  #   ports:
  #     - 9200:9200
  #   healthcheck:
  #     test: ["CMD-SHELL", "curl --silent --fail localhost:9200/_cluster/health || exit 1"]
  #     interval: 30s
  #     timeout: 10s
  #     retries: 3
  #     start_period: 40s
  #   environment:
  #     - discovery.type=single-node
  #     - bootstrap.memory_lock=true
  #     - "ES_JAVA_OPTS=-Xms4096m -Xmx4096m"
  #     - thread_pool.write.queue_size=1000
  #     - thread_pool.index.queue_size=1000
  #   ulimits:
  #     memlock:
  #       soft: -1
  #       hard: -1
  #   volumes:
  #     - data:/usr/share/elasticsearch/data
  oap:
    image: apache/skywalking-oap-server:8.1.0-es6
    container_name: oap
    # depends_on:
    #   - elasticsearch
    # links:
    #   - elasticsearch
    restart: always
    ports:
      - 11800:11800
      - 12800:12800
    healthcheck:
      test: ["CMD-SHELL", "/skywalking/bin/swctl"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    environment:
      SW_STORAGE: elasticsearch
      SW_STORAGE_ES_CLUSTER_NODES: elasticsearch:9200
      JAVA_OPTS: "-Xms2048m -Xmx2048m" 
  ui:
    image: apache/skywalking-ui:8.1.0
    container_name: ui
    depends_on:
      - oap
    links:
      - oap
    restart: always
    ports:
      - 8090:8080
    environment:
      SW_OAP_ADDRESS: oap:12800

volumes:
  data:
    driver: local

networks:
  default:
    name: datahub_network  

```
 

## Set up for Docker Images


## Understand the Metrics