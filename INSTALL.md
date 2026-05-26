# Native OpenSSL Support

This plugin supports native OpenSSL acceleration via [netty-tcnative](https://github.com/netty/netty-tcnative) (openssl-dynamic module), which dynamically links the system's `libssl`/`libcrypto` and `libapr-1`.

## Default Behavior

By default, `build.gradle` uses the official Maven Central release (`2.0.77.Final`).  
The pre-built native jars are linked against **OpenSSL 1.1.x** (`libssl.so.1.1`).

- If your system has OpenSSL 1.1.x → works out of the box.
- If your system has OpenSSL 3.x (`libssl.so.3`) → you must build netty-tcnative from source (see below).

## Building netty-tcnative for OpenSSL 3.x

### Prerequisites

```bash
# Debian/Ubuntu
sudo apt-get install -y libapr1-dev libssl-dev cmake ninja-build golang-go

# Verify OpenSSL version
openssl version
# Expected: OpenSSL 3.x.x
```

### Compile from Source

```bash
git clone https://github.com/netty/netty-tcnative.git
cd netty-tcnative
git checkout netty-tcnative-parent-2.0.77.Final   # match the version in build.gradle

# Build only the openssl-dynamic module and install to local Maven repo
mvn install -pl openssl-dynamic -am -DskipTests \
    -Dnative.build.dir=target/native-build \
    -Dos.detected.classifier=linux-x86_64

# The artifact will be installed as:
#   io.netty:netty-tcnative:2.0.77.Final:linux-x86_64
#   io.netty:netty-tcnative-classes:2.0.77.Final
```

> **Note**: If you want a SNAPSHOT version (e.g., building from `main` branch), update the version in `build.gradle` accordingly and add `mavenLocal()` to the repositories block.

### Verify

After building, confirm it links against your system OpenSSL:
```bash
ldd ~/.m2/repository/io/netty/netty-tcnative/2.0.77.Final/netty-tcnative-2.0.77.Final-linux-x86_64.jar
# Or extract the .so and check:
unzip -o ~/.m2/repository/io/netty/netty-tcnative/2.0.77.Final/netty-tcnative-2.0.77.Final-linux-x86_64.jar META-INF/native/* -d /tmp/tcnative
ldd /tmp/tcnative/META-INF/native/libnetty_tcnative_linux_x86_64.so
# Should show: libssl.so.3 => /lib/x86_64-linux-gnu/libssl.so.3
```

## Runtime Configuration

Native OpenSSL is enabled when the following system property is set:

```
-Dopensearch.unsafe.use_netty_default_allocator=true
```

The plugin will automatically detect OpenSSL availability. You can also control it via `opensearch.yml`:

```yaml
plugins.security.ssl.transport.enable_openssl_if_available: true
plugins.security.ssl.http.enable_openssl_if_available: true
```

When enabled, startup logs will show:
```
TLS Transport Provider: OPENSSL
TLS HTTP Provider: OPENSSL
OpenSSL <version> available
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `OpenSsl.isAvailable() = false` | Native lib not found or ABI mismatch | Rebuild netty-tcnative from source against your system OpenSSL |
| `libssl.so.1.1: cannot open` | Pre-built jar expects OpenSSL 1.1 | Install OpenSSL 1.1 or rebuild from source |
| `libapr-1.so.0: cannot open` | APR not installed | `apt install libapr1` |
| Falls back to JDK SSL | Missing system property | Add `-Dopensearch.unsafe.use_netty_default_allocator=true` |
