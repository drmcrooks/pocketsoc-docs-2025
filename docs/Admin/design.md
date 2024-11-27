# Design Overview

This is how the PocketSOC architecture works:

- Everything is on a private network
- Small amount of traffic: lets us focus on things that we trigger
![Image description](screenshot.png)


- Main communication is between client and webserver
    - The only command needed here on client is curl webserver

- Traffic is routed through router which mirrors it to zeek
- Zeek runs in "standalone" mode, which uses one core for processing and reduces other complexity
    - good enough for our purposes!
    - zeek here uses the pre-packaged rocky 9 binaries
- Zeek also runs filebeat which ships logs, in JSON format, to logstash

- Logstash then processes the logs and ships them to opensearch

- OpenSearch and OpenSearch dashboards use the basic example from opensearch.org
    - Don't use in production!
- Misp uses JISC docker deployment

# Architecture

The PocketSOC architecture includes several components:

![Image description](architech.png)