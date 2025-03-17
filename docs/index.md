# Trident - TRON Java SDK

## Overview

Trident is a lightweight Java SDK for interacting with the TRON blockchain. It provides a simple and efficient way to integrate TRON functionality into your Java applications.

Trident includes three main modules:

| Module | Description |
|--------|-------------|
| trident-core | Wrapping functions for easily interacting with TRON system and smart contracts |
| trident-abi | Datatypes and ABI encoders/decoders |
| trident-utils | Tools including encryption, conversion, etc. |

## Features

- Complete implementation of TRON's gRPC interfaces
- Smart contract deployment and interaction
- Wallet key management and address utilities
- Transaction building and signing
- TRC10/TRC20/TRC721 token support

## Install

Trident is compiled with Java 1.8.

### Version 0.9.2 and Later

#### Gradle

```groovy
implementation("io.github.tronprotocol:trident:0.9.2")  // Check Maven Central for newer versions
```

#### Maven

```xml
<dependency>
    <groupId>io.github.tronprotocol</groupId>
    <artifactId>trident</artifactId>
    <version>0.9.2</version>  <!-- Check Maven Central for newer versions -->
</dependency>
```

### Previous Versions (< 0.9.2)

For versions before 0.9.2, you'll need to build from source:

1. Clone the repository from GitHub
2. Build the project with required versions:
    - Java: 1.8.0_231
    - Gradle: 5.6.4
3. Configure your project dependencies:

```groovy
dependencies {
    // protobuf & grpc
    implementation 'com.google.protobuf:protobuf-java:3.11.0'
  
    implementation 'io.grpc:grpc-netty-shaded:1.31.0'
    implementation 'io.grpc:grpc-netty:1.31.0'
    implementation 'io.grpc:grpc-protobuf:1.31.0'
    implementation 'io.grpc:grpc-stub:1.31.0'
  
    implementation "org.bouncycastle:bcprov-jdk15on:1.68"

    implementation fileTree(dir:'../core')
    implementation fileTree(dir:'../utils')
    implementation fileTree(dir:'../abi')
    //if you are using the *.jar files, ues the following line
    implementation fileTree(dir:'your path', include: '*.jar')

    implementation 'com.google.guava:guava:28.0-jre'
}
```

!!! warning "Version Notice"
    We strongly recommend using version 0.9.2 or later, as newer versions are more stable and feature-rich, with all modules combined into a single package.

## Integrity Check

Starting from version 0.9.2, releases are published to Maven repository and signed with the GPG key:

```
pub: 3149 FCA5 6377 2D11 2624 9C36 CC3F 8CEA 7B0C 74D6
uid: buildtrident@tron.network
```

!!! tip
    For security verification of the package, please refer to our [Security Guide](security/gpg-verification.md).

## 🚀 Start Building with Trident

Ready to explore Trident? Check out our [Quick Start Guide](quickstart/getting-started.md) to begin building on TRON blockchain!
Let’s go!
