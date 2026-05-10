# Overview

This contains Animal Sniffer Java API signatures for different java profiles:

* CDC Foundation profile (JSR 219) API signatures with signature release: cdc11fp
* CLDC Foundation profile (JSR 139) API Signatures release: cldc11
* CDC Foundation profile (JSR 219) with base JSR 172 and JSR 280 for XML API signatures release: cdc11fpxml

How to use with Animal Sniffer Maven plugin for example against cdc11fp:

<!-- Permits to check against against certain API compatibility levels -->

<plugin>
  <groupId>org.codehaus.mojo</groupId>
  <artifactId>animal-sniffer-maven-plugin</artifactId>
  <version>1.27</version>
  <configuration>
    <signature>
         <groupId>com.optimasc.signatures</groupId>
         <artifactId>cdc11fp</artifactId>
         <version>1.0.0-SNAPSHOT</version>
    </signature>
  </configuration>
<executions>
 <execution>
    <id>check-api</id>
    <phase>test</phase>
    <goals>
          <goal>check</goal>
    </goals>
  </execution>
</executions>
</plugin>


This is provided in Public Domain.
