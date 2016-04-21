# Running px-lite with Docker Compose

You can start `px-lite` with `docker-compose` as follows:

```

# git clone https://github.com/portworx/px-lite.git
# cd px-lite/quick-start
# docker-compose run portworx -daemon --kvdb=http://myetcd.example.com:4001 --clusterid=YOUR_CLUSTER_ID --devices=/dev/xvdi
```

If you have a custom [px configuration file] (https://github.com/portworx/px-lite/edit/master/quick-start/config.json) at `/etc/pwx/config.json`, you can simply start px-lite as follows:

```
# docker-compose up -d 
```

You now have a scale-out storage cluster for containers. Continue with the [mysql example](https://github.com/portworx/px-lite/blob/master/examples/mysql.md).

As you use PX-Lite, please share your feedback and ask questions. Find the team on [Google Groups](https://groups.google.com/forum/#!forum/portworx).
