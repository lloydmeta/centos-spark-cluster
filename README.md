# Spark + Zookeeper CentOS virtual cluster

## Tips

1: `_JAVA_OPTIONS=-Djava.net.preferIPv4Stack=true` when running the driver program to avoid Host <-> Guest shenanigans
2. Make sure you can, from inside a guest, ping your host at whatever `$ hostname` returns
