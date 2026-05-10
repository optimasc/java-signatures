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

### 4. Sonatype Central account

Required only for publishing to Maven Central. Not needed for local builds, installs,
or Git tagging.

1. Create an account at <https://central.sonatype.com>.
2. Verify ownership of the `com.optimasc` namespace (one-time).
3. Generate a user token under **Account ? Generate User Token**. Save the
   resulting `username` and `password` values — they go into `settings.xml` below.

### 5. Git access to GitHub

The `developerConnection` URL in the parent POM uses SSH. Make sure your SSH key
is configured for GitHub:

```bash
ssh -T git@github.com
```

You should see a "Hi <username>! You've successfully authenticated" message.

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

Building Procedure
------------------

The following steps shall be executed to prepare and deploy a release. Complete the
Build Procedure above before proceeding.

1. Execute `mvn clean`.
2. Execute `mvn package` to build locally.
3. Execute `mvn install` to install the signatures into the local repository and
  `~/.m2/repository/com/optimasc/signatures/` and can be consumed as `<signature>`
   dependencies of the `animal-sniffer-maven-plugin` in other projects.

Release Procedure
------------------

### Tagging a release in Git

Tagging is handled by `maven-release-plugin`. The flow strips `-SNAPSHOT`, commits
the POMs, creates a Git tag, pushes the tag, and bumps to the next development
`-SNAPSHOT`. **No artifacts are published to Maven Central in this step** — it's
purely a Git operation.

### Pre-flight checklist

Before invoking the release:

- Working tree is clean (`git status` shows no changes).
- You are on the branch you intend to release from (typically `main`).
- All previous release artifacts (`release.properties`, `pom.xml.releaseBackup`)
  are absent. Run `mvn release:clean` if a previous attempt left them behind.
- GPG key is available and `settings.xml` has the `627E3C7E` server entry
  (the release process runs `verify`, which signs).
- Access to GitHub is working.

### Run the tag

```bash
mvn release:prepare
```

You will be prompted three times (defaults are usually correct):

1. **Release version** — strips `-SNAPSHOT` (e.g. `1.1.0-SNAPSHOT` ? `1.1.0`).
2. **Tag name** — defaults to `v1.1.0` based on the `tagNameFormat` configured
   in the parent POM.
3. **Next development version** — defaults to the next patch SNAPSHOT
   (e.g. `1.1.1-SNAPSHOT`).

After the prompts the plugin commits the version changes, creates the tag locally,
and pushes both the commits and the tag to `origin`. **Stop here — do not run
`release:perform` unless you want to publish to Maven Central** (see the next
section).

The plugin leaves a `release.properties` file and a few `pom.xml.releaseBackup`
files in your tree. These are needed if you later want to run `release:perform`
against this tag. If you don't intend to publish, you can clean them up with
`mvn release:clean`.

### Recovering from a failed `release:prepare`

* Failed before commit/tag was created : `mvn release:rollback`
* Tag was created locally but not pushed : `git tag -d v1.1.0`, then `mvn release:rollback`
* Tag was already pushed when failure occurred : `git push --delete origin v1.1.0`, then `mvn release:rollback`
* Leftover `release.properties` / backup POMs : `mvn release:clean` |


## Publishing to Maven Central (separately, on demand)

Publishing is a deliberate, separate step from tagging. Run it whenever you decide
the tagged version should be made public on Maven Central — this can be immediately
after `release:prepare`, days later, or never.

There are two ways to publish, depending on your situation.

### Option A — Publish a freshly tagged release

If `release.properties` is still present from a recent `release:prepare`, the
release plugin can check out the tag for you and run `mvn deploy` against it
automatically:

```bash
mvn release:perform
```

This is the cleanest option when publishing right after tagging. It checks out the
tag in a temporary working copy, runs `mvn deploy` against it, and the
`central-publishing-maven-plugin` uploads the signed artifacts to Sonatype Central.

### Option B — Publish an older tag (or any specific commit)

If `release.properties` is gone (you ran `release:clean`, switched machines,
released long ago, etc.), check out the tag manually and run `deploy`:

```bash
git checkout v1.1.0
mvn clean deploy
git checkout main
```

The signed artifacts are uploaded to Sonatype Central by the
`central-publishing-maven-plugin`.

### After the upload

The Sonatype Central deployment may need a manual release click in the portal,
depending on plugin defaults. Check the deployment status at
<https://central.sonatype.com/publishing/deployments> and release it manually if
it's in a `VALIDATED` state awaiting confirmation.

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
