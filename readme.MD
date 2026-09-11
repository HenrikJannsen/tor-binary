# Tor Binary files

Package native Tor files from [Tor Browser project](https://www.torproject.org/) expert bundles in a way that can be used by Java projects. Use the SHA256 hashes for checksum verification.

# Process

1. Fetch Tor expert bundles in parallel
2. Verify checksum
3. Package the `tor/` directory from each bundle as `tor.tar.xz`
4. Extract geoip files from the Linux x86_64 expert bundle

# Update to new version

1. Replace `torbrowser.version` with the target Tor Browser bundle version in the root [Maven file](pom.xml)
2. Find out which `tor binary version` is used in that Tor Browser release. Use that as the Maven project version.
3. Set `revision` to the `tor-binary version` in the root [Maven file](pom.xml)
4. Download the complete `sha256sums-signed-build.txt` for the new version from `torbrowser.checksum.source.url`, preserve every upstream entry, and add its source URL as the first comment in [the tracked checksum manifest](tor-binary-resources/checksums/sha256sums-signed-build.txt).
5. Verify the Tor expert bundle PGP signatures with Maven:
   ```
   mvn -N -Pcheck-pgp-signatures verify
   ```

The version-independent manifest filename makes future updates a one-file replacement. Keeping all upstream entries makes the tracked file easy to compare with its source, while the build requires an exact manifest entry and matching SHA-256 digest for every downloaded expert bundle. The source comment is intentionally the only line added to the upstream contents. For the current version, the [Tor Browser 15.0.22 checksum list](https://archive.torproject.org/tor-package-archive/torbrowser/15.0.22/sha256sums-signed-build.txt) is the source of truth. Verify its detached signature before updating the tracked manifest, then verify the bundles with:

```
curl --fail --remote-name-all \
    https://archive.torproject.org/tor-package-archive/torbrowser/15.0.22/{sha256sums-signed-build.txt,sha256sums-signed-build.txt.asc}
gpg --verify sha256sums-signed-build.txt.asc sha256sums-signed-build.txt
mvn -N -Pcheck-pgp-signatures verify
mvn clean install
```

The first two commands authenticate the downloaded checksum list before it is copied into the repository. The Maven signature profile independently verifies each expert bundle's detached signature; `mvn clean install` then checks every bundle against the pinned manifest before packaging it.

After changing the two version properties, run `mvn clean install` to confirm that all platform artifacts and the GeoIP artifact can be built.

Tor Browser versions can be found here: https://archive.torproject.org/tor-package-archive/torbrowser/[torbrowser.version]

# Pre-requisites

- GPG, when checking PGP signatures with Maven or `build.xml`
- tar with gzip/xz support
- network access to `archive.torproject.org` and `keys.openpgp.org` for PGP signature checks


# Install
Change in pom to get the desired Tor Browser bundle version

```<torbrowser.version>your TorBrowserBundle version here</torbrowser.version>```

run ```mvn install```

alternatively, you can source jitpack.io:

Maven:
```
    <repositories>
        <repository>
            <id>jitpack.io</id>
            <url>https://jitpack.io</url>
        </repository>
        ...
    </repositories>
```

Gradle:
```
    repositories {
        maven { url 'https://jitpack.io' }
        ...
    }
```


# Usage

Each platform artifact is a JAR containing `native/.../tor.tar.xz`:

```
<dependency>
    <groupId>com.github.bisq-network.tor-binary</groupId>
    <artifactId>tor-binary-linux32</artifactId>
    <version>${tor.version}</version>
</dependency>
```
```
<dependency>
    <groupId>com.github.bisq-network.tor-binary</groupId>
    <artifactId>tor-binary-linux64</artifactId>
    <version>${tor.version}</version>
</dependency>
```
```
<dependency>
    <groupId>com.github.bisq-network.tor-binary</groupId>
    <artifactId>tor-binary-macos</artifactId>
    <version>${tor.version}</version>
</dependency>
```
```
<dependency>
    <groupId>com.github.bisq-network.tor-binary</groupId>
    <artifactId>tor-binary-macos-aarch64</artifactId>
    <version>${tor.version}</version>
</dependency>
```
```
<dependency>
    <groupId>com.github.bisq-network.tor-binary</groupId>
    <artifactId>tor-binary-windows</artifactId>
    <version>${tor.version}</version>
</dependency>
```
```
<dependency>
    <groupId>com.github.bisq-network.tor-binary</groupId>
    <artifactId>tor-binary-windows64</artifactId>
    <version>${tor.version}</version>
</dependency>
```

you may want to unpack these dependencies if required using
```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
        <execution>
            <id>copy</id>
            <phase>generate-resources</phase>
            <goals>
                <goal>unpack</goal>
            </goals>
            <configuration>
                <artifactItems>
                    <artifactItem>
                        <groupId>com.github.bisq-network.tor-binary</groupId>
                        <artifactId>tor-binary-linux32</artifactId>
                        <version>${tor.version}</version>
                        <overWrite>false</overWrite>
                        <includes>native/linux/x86/tor.tar.xz</includes>
                        <outputDirectory>${project.build.directory}/classes</outputDirectory>
                    </artifactItem>
                 </artifactItems>
            </configuration>
        </execution>
    </executions>
</plugin>
```
