# Summary

This directory contains a simple hello world wildfly app created from the [wildfly-getting-started-archetype](TODO) archetype.

## Setup

* Install java

via [sdkman](https://sdkman.io/install/):

```shell
# skdman requires zip / unzip
sudo apt update
sudo apt install unzip
sudo apt install zip

# install sdkman
curl -s "https://get.sdkman.io" | bashsdk
```

install java 17 per [wildfly](https://www.wildfly.org/get-started/) guide:

```shell
sdk install java 17.0.17-tem
```

* Install [maven](https://maven.apache.org/install.html#sdkman) using sdkman:

```shell
sdk install maven
```

verify:

```shell
jberg@berg-home:~/code/injector-playground$ java -version
openjdk version "17.0.17" 2025-10-21
OpenJDK Runtime Environment Temurin-17.0.17+10 (build 17.0.17+10)
OpenJDK 64-Bit Server VM Temurin-17.0.17+10 (build 17.0.17+10, mixed mode, sharing)
jberg@berg-home:~/code/injector-playground$ mvn -version
Apache Maven 4.0.0-rc-5 (fb3ecaef88106acb40467a450248dfdbd75f3b35)
Maven home: /home/jberg/.sdkman/candidates/maven/current
Java version: 17.0.17, vendor: Eclipse Adoptium, runtime: /home/jberg/.sdkman/candidates/java/17.0.17-tem
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "6.8.0-88-generic", arch: "amd64", family: "unix"
```

## Run

* Build:

```shell
mvn package verify
```

* Run:

```shell
# the -b and -bmanagement arguments are optional. i added them so I could access from a separate machine on my LAN.
# optionally add 
./target/server/bin/standalone.sh -b=0.0.0.0 -bmanagement=0.0.0.0
# optionally, run in background
# nohup ./target/server/bin/standalone.sh -b=0.0.0.0 -bmanagement=0.0.0.0 > /dev/null 2>&1 &
```

Optionally, run with javaagent manually attached by running the following before the run command:

```shell
# download javaagent
mkdir target/javaagent && curl -L -O --output-dir target/javaagent https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar

# set grafana api key
GRAFANA_API_KEY=<insert_api_key>

# set env vars to use java agent and export data to grafana
export JAVA_TOOL_OPTIONS="-javaagent:$(pwd)/target/javaagent/opentelemetry-javaagent.jar" \
&& export OTEL_RESOURCE_ATTRIBUTES="service.name=wildfly-app" \
&& export OTEL_EXPORTER_OTLP_ENDPOINT="https://otlp-gateway-prod-us-east-2.grafana.net/otlp" \
&& export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic ${GRAFANA_API_KEY}" \
&& export OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf"
```

* Open the app in the browser:

* http://localhost:8080
* http://localhost:9990/management (credentials: TODO)

* Generate simple load:

```shell
watch curl localhost:8080
```

* Stop:

If running in foreground, stop using ctl+c (or equivalent). If running in background, kill the process listening on port 8080:

```shell
# if running in background thread
kill $(lsof -t -i:8080)
```

## Appendix: create wildfly app

* Create wildfly app

for some reason the mvn archetype was not able to download the catalog from maven. workaround:

copy archetype catalog to local maven repo:

```shell
curl -o ~/.m2/repository/custom-archetype-repo/archetype-catalog.xml https://repo.maven.apache.org/maven2/archetype-catalog.xml
curl -o ~/.m2/repository/custom-archetype-repo/archetype-catalog.xml.md5 https://repo.maven.apache.org/maven2/archetype-catalog.xml.md5
curl -o ~/.m2/repository/custom-archetype-repo/archetype-catalog.xml.sha1 https://repo.maven.apache.org/maven2/archetype-catalog.xml.sha1
```

update mvn settings to include a custom profile to read from the local repository:

```shell
touch ~/.m2/settings.xml
vim ~/.m2/settings.xml
```

set `~/.m2/settings.xml` to:

```xml
<settings>
    <profiles>
        <profile>
            <id>custom-archetype-profile</id>
            <repositories>
                <repository>
                    <id>archetype</id>
                    <url>file://${user.home}/.m2/repository/custom-archetype-repo</url>
                </repository>
            </repositories>
        </profile>
    </profiles>
    <activeProfiles>
        <activeProfile>custom-archetype-profile</activeProfile>
    </activeProfiles>
</settings>
```

generate sample project

```shell
mvn -U archetype:generate \
    -DarchetypeGroupId=org.wildfly.archetype \
    -DarchetypeArtifactId=wildfly-getting-started-archetype
```

(the output of this is checked into vcs)
