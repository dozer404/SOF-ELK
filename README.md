# SOF-ELK on TrueNAS

## Purpose

This project provides a containerized deployment of the [SOF-ELK](https://github.com/philhagen/sof-elk) platform for TrueNAS. It is intended to make it easier to run SOF-ELK as a persistent, self-contained application stack using Docker Compose-style services backed by a TrueNAS dataset.

SOF-ELK is commonly used for security operations, incident response, log analysis, and forensic timeline review. This deployment packages the supporting Elastic Stack services and filesystem-based ingestion paths so logs can be dropped into mounted directories on TrueNAS and processed by SOF-ELK.

## What This Deploys

The compose file deploys the following services:

| Service | Purpose |
| --- | --- |
| `sof-elk-init` | Creates the required directory structure, clones or updates the SOF-ELK repository, checks out the configured SOF-ELK version, writes the Filebeat configuration, and sets permissions. |
| `sof-elk-elasticsearch` | Stores indexed event data for search and analysis. |
| `sof-elk-kibana` | Provides the web interface for exploring indexed logs and dashboards. |
| `sof-elk-logstash` | Runs the SOF-ELK Logstash pipelines and receives events from Filebeat. |
| `sof-elk-filebeat` | Watches local ingest directories and forwards files to Logstash. |

## TrueNAS Dataset Layout

Before deployment, replace the placeholder dataset name in the YAML file:

```yaml
/mnt/YOUR_DATASET_NAME/ix-applications/sof-elk
```

Use the name of the TrueNAS dataset where you want SOF-ELK data, configuration, repositories, ingest folders, and persistent service data to live.

The deployment creates and uses a directory layout similar to this:

```text
/mnt/YOUR_DATASET_NAME/ix-applications/sof-elk/
├── elasticsearch-data/
├── filebeat/
│   └── filebeat.yml
├── filebeat-data/
├── geoip/
├── ingest/
│   ├── aws/
│   ├── azure/
│   ├── gcp/
│   ├── gws/
│   ├── hayabusa/
│   ├── httpd/
│   ├── kape/
│   ├── kubernetes/
│   ├── microsoft365/
│   ├── nfarch/
│   ├── passivedns/
│   ├── plaso/
│   ├── syslog/
│   ├── volatility/
│   └── zeek/
├── kibana-data/
├── logstash-data/
└── sof-elk/
```

## Exposed Ports

| Port | Service | Description |
| --- | --- | --- |
| `5601/tcp` | Kibana | SOF-ELK web interface. |
| `9200/tcp` | Elasticsearch | Elasticsearch HTTP API. |
| `9300/tcp` | Elasticsearch | Elasticsearch transport port. |
| `5044/tcp` | Logstash | Beats input used by Filebeat. |
| `9995/udp` | Logstash | UDP listener for supported SOF-ELK ingest pipelines. |

## Deployment Steps

1. Create or choose a TrueNAS dataset for the deployment.
2. Edit the YAML file and replace `YOUR_DATASET_NAME` with your actual dataset name.
3. Deploy the stack through the TrueNAS custom app or Docker Compose workflow.
4. Wait for `sof-elk-init` to finish successfully.
5. Confirm that Elasticsearch, Logstash, Filebeat, and Kibana start without errors.
6. Open Kibana at:

```text
http://<TRUENAS_HOST_IP>:5601
```

## Log Ingestion

Filebeat is configured to recursively watch the SOF-ELK ingest directories under:

```text
/mnt/YOUR_DATASET_NAME/ix-applications/sof-elk/ingest/
```

Place supported logs into the appropriate ingest directory. For example:

```text
ingest/aws/
ingest/azure/
ingest/gcp/
ingest/kape/
ingest/plaso/
ingest/syslog/
ingest/zeek/
```

Filebeat forwards matching files to Logstash on port `5044`, where the SOF-ELK pipelines parse and enrich the data before indexing it into Elasticsearch.

## GeoIP Databases

The Logstash container checks for the following optional GeoIP database files:

```text
geoip/GeoLite2-ASN.mmdb
geoip/GeoLite2-City.mmdb
geoip/GeoLite2-Country.mmdb
```

Place those files in:

```text
/mnt/YOUR_DATASET_NAME/ix-applications/sof-elk/geoip/
```

The stack can start without them, but GeoIP enrichment may be limited until the databases are added.

## Persistence

This deployment stores service data on the TrueNAS dataset so application state can survive container restarts and upgrades.

Persistent paths include:

```text
elasticsearch-data/
kibana-data/
logstash-data/
filebeat-data/
sof-elk/
ingest/
geoip/
```

## Resource Notes

The compose configuration assigns JVM heap settings for Elasticsearch and Logstash:

```yaml
ES_JAVA_OPTS: '-Xms4g -Xmx4g'
LS_JAVA_OPTS: '-Xms2g -Xmx2g'
```

Make sure the TrueNAS host has enough available memory for the Elastic Stack, TrueNAS itself, and any other running applications. Adjust the heap values if needed for your environment.

## Security Notes

This deployment disables Elastic Stack security features for ease of local deployment:

```yaml
xpack.security.enabled: 'false'
XPACK_SECURITY_ENABLED: 'false'
```

Do not expose Kibana, Elasticsearch, Logstash, or Filebeat ports directly to the internet. Restrict access to trusted networks, use firewall rules, and consider adding authentication, reverse proxy controls, or VPN-only access for production or shared environments.

## Operational Commands

Check container status:

```bash
docker ps
```

View logs for a service:

```bash
docker logs sof-elk-logstash
docker logs sof-elk-filebeat
docker logs sof-elk-kibana
docker logs sof-elk-elasticsearch
```

Restart a service:

```bash
docker restart sof-elk-logstash
```

## Troubleshooting

### Kibana does not load

Confirm Elasticsearch is healthy and reachable:

```bash
curl http://<TRUENAS_HOST_IP>:9200
```

Then check Kibana logs:

```bash
docker logs sof-elk-kibana
```

### Logstash exits because the data directory is not writable

Verify permissions on:

```text
/mnt/YOUR_DATASET_NAME/ix-applications/sof-elk/logstash-data
```

The init container attempts to set ownership and permissions automatically, but TrueNAS dataset ACLs may still need adjustment.

### Logs are not appearing in Kibana

Check the following:

- Files are placed in the correct `ingest/` subdirectory.
- Filebeat is running and can connect to Logstash.
- Logstash is running and listening on port `5044`.
- The SOF-ELK pipeline supports the log type you are ingesting.

Useful log commands:

```bash
docker logs sof-elk-filebeat
docker logs sof-elk-logstash
```

## Disclaimer

This is a containerized TrueNAS deployment wrapper for SOF-ELK. SOF-ELK itself is maintained by its upstream project. Review upstream documentation for supported ingest formats, pipeline behavior, and version-specific guidance.
