# Building, Installing, and Deploying

This document describes how to build the Animal Sniffer signature artifacts locally,
install them into your local Maven repository, deploy them to Maven Central, and tag
a release in Git.

## Project layout

The project is a Maven multi-module build:

```
java-signatures/
+ pom.xml          parent (aggregator)
|- cdcfp/          cdc11fp        — CDC Foundation Profile 1.1.2
|- cldc/           cldc11         — CLDC Foundation Profile (JSR 139)
|- cdcxml/         cdc11fpxml     — CDC FP 1.1.2 + JSR 172 + JSR 280
```

Running any Maven command from the project root applies it to all three modules.

---

## Prerequisites

### 1. Tooling

* Any modern JDK 8+; only used to run Maven
* Maven 3.6+ 
* Git
* GPG 2.x 

Verify each is on your `PATH`:

```bash
java -version
mvn -version
git --version
gpg --version
```

### 2. Reference JAR files (Windows paths)

Each child module's `animal-sniffer-maven-plugin` execution references reference-API
JARs at hardcoded Windows paths. These files must exist before a build can succeed:

```
C:\PortableApps\Java\libs\cdc_1.1.jar
C:\PortableApps\Java\libs\cldc_1.1.jar
C:\PortableApps\Java\libs\fp_1.1.jar
C:\PortableApps\Java\libs\jsr172_1.0-base.jar
C:\PortableApps\Java\libs\jsr280_1.0.jar
```

If you build on a non-Windows host, edit the `<javaHomeClassPath>` entries in each
child POM accordingly.

### 3. GPG key

Artifacts are signed during the `verify` phase using key ID `627E3C7E`. You must have
this key (or substitute your own — see below) in your GPG keyring, and the public key
must be published to a public keyserver before Maven Central will accept the artifacts.

Check the key is present:

```bash
gpg --list-secret-keys 627E3C7E
```

Publish the public key (one-time):

```bash
gpg --keyserver keyserver.ubuntu.com --send-keys 627E3C7E
```

To use a different key, override `gpg.keyname` on the command line:

```bash
mvn verify -Dgpg.keyname=YOURKEYID
```

---

## `~/.m2/settings.xml`

Maven reads credentials and passphrases from `~/.m2/settings.xml` (`%USERPROFILE%\.m2\settings.xml`
on Windows). Create or edit the file as follows:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              https://maven.apache.org/xsd/settings-1.0.0.xsd">

  <servers>

    <!-- Sonatype Central credentials (user token, NOT your portal password) -->
    <server>
      <id>central</id>
      <username>YOUR_CENTRAL_TOKEN_USERNAME</username>
      <password>YOUR_CENTRAL_TOKEN_PASSWORD</password>
    </server>

    <!-- GPG passphrase. The id MUST match the gpg.keyname property
         in the parent pom (627E3C7E) because the gpg plugin is configured
         with <passphraseServerId>${gpg.keyname}</passphraseServerId>. -->
    <server>
      <id>627E3C7E</id>
      <passphrase>YOUR_GPG_PASSPHRASE</passphrase>
    </server>

  </servers>

</settings>
```

### Notes on the entries

- The `<id>central</id>` value matches the `<publishingServerId>central</publishingServerId>`
  in the parent POM. Don't rename it.
- The `<id>627E3C7E</id>` value matches `${gpg.keyname}`. If you build with a different
  key (`-Dgpg.keyname=...`), add a corresponding `<server>` entry with that id.
- If you prefer not to store the GPG passphrase in plain text, omit the `<passphrase>`
  element entirely and rely on `gpg-agent` to prompt you (works on most modern systems).
- Sensitive values can be encrypted using
  [Maven password encryption](https://maven.apache.org/guides/mini/guide-encryption.html)
  if you want to avoid plain text entirely.

### File permissions

On Linux and macOS, restrict access:

```bash
chmod 600 ~/.m2/settings.xml
```

---

## Build, install, deploy

All commands are run from the project root unless noted otherwise.

### Build only (compile and generate signatures)

```bash
mvn clean package
```

Each module produces `target/<artifactId>-<version>.signature` plus signed `.asc`
companion files. Nothing is installed or published.

### Install into the local repository

```bash
mvn clean install
```

After this, the three signature artifacts are available in `~/.m2/repository/com/optimasc/signatures/`
and can be consumed as `<signature>` dependencies of the `animal-sniffer-maven-plugin`
in other projects.

### Deploy to Maven Central

```bash
mvn clean deploy
```

This runs the full lifecycle through the `deploy` phase, which invokes
`central-publishing-maven-plugin`. The plugin uploads the signed artifacts to
Sonatype Central and (because `<autoPublish>` is at its default) waits for validation.

You can monitor and manually release the deployment from the Central portal at
<https://central.sonatype.com/publishing/deployments>.

### Build a single module

```bash
mvn -pl cdcfp clean install
```

Use `-pl <module>` (project list) to operate on one module at a time. Add `-am`
(also-make) if you ever introduce inter-module dependencies.

---

## Releasing (tagging in Git)

Releases are managed by `maven-release-plugin`. The flow strips `-SNAPSHOT`, commits
the POMs, creates a Git tag, pushes the tag, and bumps to the next development version.

### Prerequisites for a release

- Working tree is clean (`git status` shows no changes).
- You are on the branch you intend to release from (typically `main`).
- Your SSH key is configured for GitHub (the `developerConnection` URL uses SSH).
- All previous release artifacts (`release.properties`, `pom.xml.releaseBackup`) are absent.
  Run `mvn release:clean` if a previous attempt left them behind.

### Prepare the release

```bash
mvn release:prepare
```

You will be prompted three times:

1. **Release version** — strips `-SNAPSHOT` (e.g. `1.1.0-SNAPSHOT` ? `1.1.0`).
2. **Tag name** — defaults to `v1.1.0` based on the `tagNameFormat` configured
   in the parent POM.
3. **Next development version** — defaults to the next patch SNAPSHOT (e.g. `1.1.1-SNAPSHOT`).

The plugin then commits the version changes, creates the tag locally, and pushes
both the commits and the tag to `origin`.

### Perform the release (optional — only if publishing to Central)

```bash
mvn release:perform
```

This checks out the freshly created tag and runs `mvn deploy` against it. Requires
the Sonatype Central credentials in `settings.xml` to be valid.

If you only wanted to tag in Git and don't want to publish to Central, skip this step.

### Recovering from a failed release

| Situation | Command |
|---|---|
| Failed before commit/tag was created | `mvn release:rollback` |
| Tag was created locally but not pushed | `git tag -d v1.1.0` then `mvn release:rollback` |
| Tag was already pushed | `git push --delete origin v1.1.0`, then `mvn release:rollback` |
| Leftover `release.properties` / backup POMs | `mvn release:clean` |

---

## Troubleshooting

**`gpg: signing failed: No such file or directory`** — `gpg-agent` is not running
or cannot prompt for the passphrase. Ensure the passphrase is in `settings.xml` (under
the `627E3C7E` server id) or configure `gpg-agent` and `pinentry`.

**`401 Unauthorized` from Sonatype Central** — the user token in `settings.xml` is
incorrect, expired, or you used your portal password instead of a generated user token.
Regenerate the token at <https://central.sonatype.com>.

**`Cannot run program "git"`** — Git is not on `PATH`. Required by `maven-release-plugin`.

**`The git-push command failed`** — your SSH key isn't configured for GitHub, or the
`developerConnection` URL is wrong. Test with `ssh -T git@github.com`.

**`A reference JAR was not found at C:\PortableApps\Java\libs\...`** — the reference
JARs listed under [Prerequisites](#2-reference-jar-files-windows-paths) are missing
or you're on a non-Windows host. Place the JARs at the expected paths or edit the
child POMs.
