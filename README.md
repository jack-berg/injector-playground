# Summary

This repo contains notes on getting started with the `opentelemetry-injector` in a linux environment.

To demonstrate the injector, we have a hello world [Java wildfly app](./wildfly-app).

We'll be configuring the injector to export data via OTLP to grafana cloud.

## Injector installation

* Download latest injector (currently [v0.5.0](https://github.com/open-telemetry/opentelemetry-injector/releases/tag/v0.5.0)). Replace the link in the command below with the appropriate deb / rpm package based on preferred package manager system arch. In my case, I use a `deb` package and my system arch is `amd64`.   

```shell
curl -L -O --output-dir ./out https://github.com/open-telemetry/opentelemetry-injector/releases/download/v0.5.0/opentelemetry-injector_0.5.0_amd64.deb
chmod 644 out/opentelemetry-injector_0.5.0_amd64.deb
```

* Install injector

```shell
sudo apt install ./out/opentelemetry-injector_0.5.0_amd64.deb
```

(to uninstall)

```shell
sudo apt remove opentelemetry-injector
```

* Notice the resources it installs in `/usr/lib/opentelemetry` and `/etc/opentelemetry`:

```shell
# main program and agents in /usr/lib/opentelemetry
jberg@berg-home:~/code/injector-playground$ ls -lh /usr/lib/opentelemetry
drwxr-xr-x 4 root root 4.0K Nov 25 21:01 dotnet
drwxr-xr-x 2 root root 4.0K Apr  8 18:02 jvm
-rwxr-xr-x 1 root root 2.7M Apr  7 15:59 libotelinject.so
drwxr-xr-x 3 root root 4.0K Apr  8 18:02 nodejs

# other config resources in /etc/opentelemetry
jberg@berg-home:~/code/injector-playground$ ls -lh /etc/opentelemetry/
total 12K
-rw-r--r-- 1 root root 363 Dec 17 22:42 default_auto_instrumentation_env.conf
-rwxr-xr-x 1 root root 473 Apr  7 16:00 otelinject.conf
```

* Configure global environment variables to export to grafana via OTLP:

Open `/etc/opentelemetry/default_auto_instrumentation_env.conf`:

```shell
sudo vim /etc/opentelemetry/default_auto_instrumentation_env.conf
```

Add the following content:

```
OTEL_EXPORTER_OTLP_ENDPOINT=https://otlp-gateway-prod-us-east-2.grafana.net/otlp
OTEL_EXPORTER_OTLP_HEADERS=Authorization=Basic <insert_api_key>>
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

* Add `libotelinject.so` to `/etc/ld.so.preload`. After doing this step, the injector will be invoked on every process start. If issues occur and you need the injector to stop, delete the `libotelinject.so` line from `/etc/ld.so.preload`.

```shell
sudo touch /etc/ld.so.preload
sudo sh -c "echo /usr/lib/opentelemetry/libotelinject.so >> /etc/ld.so.preload"
```

This results in `/etc/ld.so.preload`:

```shell
jberg@berg-home:~/code/injector-playground-grafana$ cat /etc/ld.so.preload
/usr/lib/opentelemetry/libotelinject.so
```

* Run wildfly app per [instructions](wildfly-app/README.md), omitting the steps to download and configure the OpenTelemetry Java Agent:

Notably, `JAVA_TOOL_OPTIONS` should not need to be set. Should see log output indicating java agent was added:

```
jberg@berg-home:~/code/injector-playground/wildfly-app$ ./target/server/bin/standalone.sh -b=0.0.0.0 -bmanagement=0.0.0.0
=========================================================================

  JBoss Bootstrap Environment

  JBOSS_HOME: /home/jberg/code/injector-playground/wildfly-app/target/server

  JAVA: /home/jberg/.sdkman/candidates/java/current/bin/java

  JAVA_OPTS:  -Djdk.serialFilter="maxbytes=10485760;maxdepth=128;maxarray=100000;maxrefs=300000" -Xms64m -Xmx512m -Djava.net.preferIPv4Stack=true -Djboss.modules.system.pkgs=org.jboss.byteman -Djava.awt.headless=true  --add-exports=java.desktop/sun.awt=ALL-UNNAMED --add-exports=java.naming/com.sun.jndi.ldap=ALL-UNNAMED --add-exports=java.naming/com.sun.jndi.url.ldap=ALL-UNNAMED --add-exports=java.naming/com.sun.jndi.url.ldaps=ALL-UNNAMED --add-exports=jdk.naming.dns/com.sun.jndi.dns=ALL-UNNAMED --add-opens=java.base/java.lang=ALL-UNNAMED --add-opens=java.base/java.lang.invoke=ALL-UNNAMED --add-opens=java.base/java.lang.reflect=ALL-UNNAMED --add-opens=java.base/java.io=ALL-UNNAMED --add-opens=java.base/java.net=ALL-UNNAMED --add-opens=java.base/java.security=ALL-UNNAMED --add-opens=java.base/java.util=ALL-UNNAMED --add-opens=java.base/java.util.concurrent=ALL-UNNAMED --add-opens=java.management/javax.management=ALL-UNNAMED --add-opens=java.naming/javax.naming=ALL-UNNAMED

=========================================================================

Picked up JAVA_TOOL_OPTIONS: -javaagent:/usr/lib/opentelemetry/javaagent.jar
OpenJDK 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended
[otel.javaagent 2026-01-14 22:57:18:242 +0000] [main] INFO io.opentelemetry.javaagent.tooling.VersionLogger - opentelemetry-javaagent - version: 2.16.0
```

If everything worked, should eventually see a service with name `JBoss Modules` in grafana.

## Injector upgrade

* Download new injector

```shell
curl -L -O --output-dir ./out https://github.com/open-telemetry/opentelemetry-injector/releases/download/v0.0.3/opentelemetry-injector_0.0.3_amd64.deb
```

* Upgrade to new file

```shell
# make executable to avoid warnings of the form:
# N: Download is performed unsandboxed as root as file '/home/jberg/code/injector-playground/out/opentelemetry-injector_0.0.2-20251216_amd64.deb' couldn't be accessed by user '_apt'. - pkgAcquire::Run (13: Permission denied)
sudo chmod 755 ./out/opentelemetry-injector_0.0.3_amd64.deb
sudo apt install ./out/opentelemetry-injector_0.0.3_amd64.deb
```

* When prompted to handle change in `/etc/opentelemetry/otelinject.conf`, make a decision based on your situation:

```shell
Configuration file '/etc/opentelemetry/otelinject.conf'
 ==> Modified (by you or by a script) since installation.
 ==> Package distributor has shipped an updated version.
   What would you like to do about it ?  Your options are:
    Y or I  : install the package maintainer's version
    N or O  : keep your currently-installed version
      D     : show the differences between the versions
      Z     : start a shell to examine the situation
```

* Re-add `libotelinject.so` to `/etc/ld.so.preload`:


```shell
sudo touch /etc/ld.so.preload
sudo sh -c "echo /usr/lib/opentelemetry/libotelinject.so >> /etc/ld.so.preload"
```

## Notes

* The injector instrumented a bunch of java processes which I did not expect. It turns out, my remote IDE setup allowing me to develop from my mac to the linux machine is implemented in java. Can use [include/exclude](https://github.com/open-telemetry/opentelemetry-injector?tab=readme-ov-file#details-on-configuring-the-program-inclusion-and-exclusion-criteria) configuration to tune what gets installed.
* At one point I updated to the latest injector version and there was a [bug in the injector code](https://github.com/open-telemetry/opentelemetry-injector/issues/173) causing all new processes to fail. It was difficult to bring my system back to stability because I wasn't even able to use vim to modify `/etc/ld.so.preload`. For some reason, while `vim` wasn't able to start due to the injector bug, `rm` was unimpacted. I restored stability with `sudo rm /etc/ld.so.preload`. Conclusion: LD_PRELOAD is a powerful lever requiring the injector to have a proportionally high quality bar.
* Naming applications will be a recurring issue. The whole point of the injector is to avoid needing to modify application code. How does `service.name` get set to a sensible value if nobody is setting it? The otel java agent has some extra logic to detect an appropriate `service.name` from various locations frameworks are known to keep metadata. What about other languages? If you see `service.name` being set to `unknown_service:<language>`, or distinct services being set to the same name, you may have to jump through some extra hoops to set the `OTEL_SERVICE_NAME=<the service name>` env var for individual processes. I opened an issue to track this [here](https://github.com/open-telemetry/opentelemetry-injector/issues/146).