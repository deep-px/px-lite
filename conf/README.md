# Sample config.json

### Explanation of the fields:
1. mgtiface - optional.  Network interface to use for management traffic.
2. dataiface - optional. Network interface to use for storage traffic.
3. loggingurl - optional.  URL to use for logging.
4. alertingurl - optional.  URL to use for alerts.
5. storage - mandatory.  Devices to use for physical storage.
6. debug_level - optional. Set to "low" for performance runs
7. kvdb - mandatory.  Key value database for configuration and node discovery.
8. clusterid - mandatory.  A unique cluster ID.  Nodes in the same cluster must have the same cluster ID.

