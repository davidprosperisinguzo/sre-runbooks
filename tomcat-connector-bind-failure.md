# Tomcat Connector Bind Failure on Shared Lab Server — Root Cause and Fix

**Environment:** RHEL/CentOS-based app server (`stapp01`), Tomcat 9 installed via `dnf`
**Service affected:** `tomcat.service`
**Impact:** Java web application (`ROOT.war`) required deployment on port 8088; Tomcat's HTTP connector silently failed to bind, despite the service reporting as running

## Symptom

Tomcat was installed, enabled, and started without any error at the command line:

```bash
sudo systemctl start tomcat
sudo systemctl enable tomcat
```

`systemctl status tomcat` reported `Active: active (running)`, with a live PID and no obvious failure at a glance. But the tail of the status output included partially truncated log lines referencing a stack trace, easy to miss if the "active (running)" line is taken at face value without reading further.

## Diagnosis Path

### 1. Don't trust "active (running)" alone, read the actual log

```bash
sudo journalctl -u tomcat --no-pager -n 100
```

This is the step that mattered. Buried in the output was the real failure:

```
SEVERE [main] org.apache.catalina.util.LifecycleBase.handleSubClassException
Failed to initialize component [Connector["http-nio-8080"]]
org.apache.catalina.LifecycleException: Protocol handler initialization failed
Caused by: java.net.BindException: Address already in use
```

The systemd unit stayed alive because the parent Java process didn't crash, it just failed to initialize its HTTP connector. This is a good example of why a service reporting "running" isn't the same as a service actually doing its job, the process can be alive and still non-functional.

### 2. Identify what was actually holding the conflicting port

The standard tool for this, `ss`, wasn't installed on this server:

```bash
ss -tulnp | grep 8080
# bash: ss: command not found
```

Attempting to install a package literally named `ss` failed, because that's not how it's packaged:
```bash
sudo dnf install ss
# Error: Unable to find a match: ss
```

`ss` ships inside the `iproute` package on RHEL/CentOS, not as a standalone package matching the binary name:
```bash
sudo dnf install iproute -y
```

With that installed:
```bash
sudo ss -tulnp | grep 8080
```
```
tcp   LISTEN 0      128    0.0.0.0:8080   0.0.0.0:*   users:(("ttyd",pid=60,fd=11))
```

### 3. Confirm what was found before doing anything about it

`ttyd` turned out to be the process powering the browser-based terminal session used to access this lab server in the first place, part of the lab platform's own infrastructure, not anything related to the actual task. It had port 8080 permanently claimed on this box. Confirming this before taking any action mattered: killing an unfamiliar process squatting on a needed port without checking what it is first risks cutting off your own access to the server.

## Root Cause

Tomcat's default connector configuration listens on port 8080. On this particular server, port 8080 was already permanently occupied by `ttyd` (the lab environment's own terminal-access process), so Tomcat's connector failed to bind at startup. The systemd service itself did not fail or restart, it simply ran with a non-functional HTTP listener, no port open, no way to reach the application, despite every surface-level indicator suggesting the service was healthy.

## Fix Applied

Since the task's actual requirement was to run Tomcat on port 8088 rather than the default 8080, changing the connector port resolved the conflict entirely, no need to touch or stop the process holding 8080:

```bash
sudo vi /etc/tomcat/server.xml
```

Changed:
```xml
<Connector port="8080" protocol="HTTP/1.1" ... />
```
to:
```xml
<Connector port="8088" protocol="HTTP/1.1" ... />
```

Deployed the application by placing `ROOT.war` directly into Tomcat's webapps directory (located via `rpm -ql tomcat | grep webapps`, since this package layout doesn't use the `/opt/tomcat` structure a manual tarball install would):

```bash
sudo cp /home/tony/ROOT.war /usr/share/tomcat/webapps/
sudo chown tomcat:tomcat /usr/share/tomcat/webapps/ROOT.war
```

Naming the file exactly `ROOT.war` matters here: Tomcat maps a file with that specific name to the root context (`/`), so the application became reachable at the base URL directly, with no additional path segment required.

Restarted and verified clean startup:
```bash
sudo systemctl restart tomcat
sudo journalctl -u tomcat --no-pager -n 50
sudo ss -tulnp | grep 8088
```

Confirmed the connector initialized without exception this time, and `java`/`tomcat` was shown actually bound to 8088.

## Key Lesson

**`systemctl status` reporting "active (running)" is not proof that a service is functioning correctly.** It confirms the process is alive, not that every component inside it initialized successfully. A connector, a listener, a subsystem can fail quietly while the parent process keeps running. `journalctl -u <service>` is what actually shows whether startup completed cleanly, not just whether the process is still there.

**Package names don't always match binary names.** Trying to `dnf install ss` failed because the tool lives inside `iproute`. When a command isn't found and the obvious install guess fails too, it's worth searching for which package actually provides that binary rather than assuming the name is a 1:1 match.

**Identify a process before acting on it, especially on shared or lab infrastructure.** Port conflicts aren't always caused by leftover application processes, sometimes the thing holding a port is the platform you're working inside of. Confirming what `ttyd` was before considering killing it avoided potentially cutting off the very session being used to fix the problem.

## Verifying the Fix Is Complete

```bash
sudo ss -tulnp | grep 8088
curl http://localhost:8088
```
Then from wherever the task actually specifies validation should happen (in this case, the jump host):
```bash
curl http://stapp01:8088
```
A successful response confirms both the connector bind and the WAR deployment are working end to end, not just that the port is open.

## General Troubleshooting Habit

When a service "starts" without an obvious error at the command line, check its actual startup log before moving on, especially for anything involving network binding. A process can report success at the systemd level while failing internally at a level systemd itself isn't aware of.