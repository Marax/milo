# Migration Guide: Eclipse Milo 0.6.7 to 1.1.1

This guide provides detailed instructions for migrating from Eclipse Milo version 0.6.7 to version 1.1.1.

## Table of Contents

1. [Overview](#overview)
2. [Breaking Changes Summary](#breaking-changes-summary)
3. [Version 1.0.0 Major Refactoring](#version-100-major-refactoring)
4. [Client API Migration](#client-api-migration)
5. [Server API Migration](#server-api-migration)
6. [Data Type Changes](#data-type-changes)
7. [Security and Certificates](#security-and-certificates)
8. [Dependencies](#dependencies)
9. [Code Examples](#code-examples)
10. [Troubleshooting](#troubleshooting)

---

## Overview

The migration from version 0.6.7 to 1.1.1 represents a **major upgrade** with significant architectural changes introduced primarily in version 1.0.0. This release includes:

- **487 commits** with changes to over **3,509 files**
- Support for **OPC UA 1.05** specification
- Upgrade to **Java 17** (minimum requirement)
- Major refactoring of client and server APIs
- New JSON encoding support
- Enhanced data type handling
- Improved security and certificate management

**Important**: This is a breaking change migration requiring code modifications.

---

## Breaking Changes Summary

### Critical Breaking Changes

1. **Java Version**: Minimum Java version upgraded from Java 8 to **Java 17**
2. **Package Structure**: Major refactoring away from "api" packages
3. **Client API**: Complete redesign of `OpcUaClient` API
4. **Server API**: Changes to `AddressSpace`, `AttributeContext`, and node handling
5. **Data Types**: `BuiltinDataType` renamed to `OpcUaDataType`
6. **Lombok Removal**: All Lombok-generated code replaced with custom implementations
7. **ExtensionObject**: Now a sealed class with new implementation
8. **ExpandedNodeId**: Completely reimplemented
9. **Subscription API**: New client subscription and monitored item API

---

## Version 1.0.0 Major Refactoring

Version 1.0.0 introduced the most significant changes. Key refactorings include:

### OPC UA 1.05 Support

```xml
<!-- Update your dependencies -->
<dependency>
    <groupId>org.eclipse.milo</groupId>
    <artifactId>milo-sdk-client</artifactId>
    <version>1.1.1</version>
</dependency>
```

### Package Structure Changes

The removal of "api" packages means imports need updating:

**Before (0.6.7):**
```java
import org.eclipse.milo.opcua.sdk.client.api.OpcUaClient;
import org.eclipse.milo.opcua.sdk.server.api.AddressSpace;
```

**After (1.1.1):**
```java
import org.eclipse.milo.opcua.sdk.client.OpcUaClient;
import org.eclipse.milo.opcua.sdk.server.AddressSpace;
```

### Transport and Stack Refactoring

The stack and transport layers underwent major refactoring in PR [#1078](https://github.com/eclipse-milo/milo/pull/1078). This affects low-level usage but most high-level APIs remain similar.

---

## Client API Migration

### 1. OpcUaClient Creation and Configuration

The `OpcUaClient` API was refactored in PR [#1126](https://github.com/eclipse-milo/milo/pull/1126).

**Before (0.6.7):**
```java
OpcUaClient client = OpcUaClient.create(
    "opc.tcp://localhost:4840",
    endpoints -> endpoints.stream()
        .filter(e -> e.getSecurityPolicyUri().equals(SecurityPolicy.None.getUri()))
        .findFirst(),
    configBuilder -> configBuilder
        .setApplicationName(LocalizedText.english("My Client"))
        .setApplicationUri("urn:my:client")
        .build()
);
```

**After (1.1.1):**
```java
OpcUaClient client = OpcUaClient.create(
    "opc.tcp://localhost:4840",
    builder -> builder
        .setApplicationName(LocalizedText.english("My Client"))
        .setApplicationUri("urn:my:client")
        .setEndpointFilter(endpoints -> endpoints.stream()
            .filter(e -> e.getSecurityPolicyUri().equals(SecurityPolicy.None.getUri()))
            .findFirst())
        .build()
);
```

### 2. Subscription API Changes

Complete redesign of subscription and monitored item API in PR [#1223](https://github.com/eclipse-milo/milo/pull/1223).

**Before (0.6.7):**
```java
UaSubscription subscription = client.getSubscriptionManager()
    .createSubscription(1000.0)
    .get();

UaMonitoredItem item = subscription.createMonitoredItem(
    new ReadValueId(nodeId, AttributeId.Value.uid(), null, QualifiedName.NULL_VALUE),
    MonitoringMode.Reporting,
    new MonitoringParameters(
        uint(1),
        1000.0,
        null,
        uint(10),
        true
    )
).get();

item.setValueConsumer(v -> {
    System.out.println("Value: " + v.getValue().getValue());
});
```

**After (1.1.1):**
```java
ManagedSubscription subscription = client.getSubscriptionManager()
    .createSubscription(1000.0)
    .get();

ManagedDataItem dataItem = subscription.createDataItem(nodeId);

dataItem.addDataValueListener(v -> {
    System.out.println("Value: " + v.getValue().getValue());
});
```

### 3. DataTypeTree and Codec Registration

New support for DataTypeTree and codec registration in PR [#1222](https://github.com/eclipse-milo/milo/pull/1222).

**After (1.1.1):**
```java
// Register custom data types
DataTypeTree dataTypeTree = DataTypeTreeBuilder.build(
    client.getAddressSpace(),
    client.getNamespaceTable()
);

client.getDynamicDataTypeManager().registerCodec(
    new CustomStructCodec(dataTypeTree)
);
```

### 4. Operation Limits Support

New support for reading operation limits in PR [#1221](https://github.com/eclipse-milo/milo/pull/1221).

**After (1.1.1):**
```java
OperationLimits operationLimits = client.getOperationLimits()
    .orElse(OperationLimits.DEFAULT);

int maxNodesPerRead = operationLimits.getMaxNodesPerRead();
```

---

## Server API Migration

### 1. AddressSpace API Changes

The `AddressSpace` was converted to a blocking API in PR [#1245](https://github.com/eclipse-milo/milo/pull/1245).

**Before (0.6.7):**
```java
// Async API
CompletableFuture<Optional<UaNode>> future = 
    addressSpace.getNodeAsync(nodeId);
```

**After (1.1.1):**
```java
// Blocking API
Optional<UaNode> node = addressSpace.getNode(nodeId);
```

### 2. Browse API Refactoring

The Browse API was refactored in PR [#1246](https://github.com/eclipse-milo/milo/pull/1246).

**After (1.1.1):**
```java
// Use ReferenceTypeTree for reference subtype checking
ReferenceTypeTree referenceTypeTree = 
    server.getAddressSpaceManager().getReferenceTypeTree();
```

### 3. AttributeContext Replaced with AccessContext

In PR [#1160](https://github.com/eclipse-milo/milo/pull/1160), `AttributeContext` was replaced with `AccessContext`.

**Before (0.6.7):**
```java
void onAttributeRead(
    AttributeContext context,
    AttributeId attributeId
) {
    // ...
}
```

**After (1.1.1):**
```java
void onAttributeRead(
    AccessContext context,
    AttributeId attributeId
) {
    // ...
}
```

### 4. AttributeDelegate Removal

`AttributeDelegate` and related code was removed in PR [#1159](https://github.com/eclipse-milo/milo/pull/1159). Use `AttributeFilter` instead:

**After (1.1.1):**
```java
AttributeFilter filter = new AttributeFilter() {
    @Override
    public Object readAttribute(
        AccessContext context,
        UaNode node,
        AttributeId attributeId
    ) {
        // Custom read logic
    }
    
    @Override
    public void writeAttribute(
        AccessContext context,
        UaNode node,
        AttributeId attributeId,
        Object value
    ) {
        // Custom write logic
    }
};

server.getAddressSpaceManager().addAttributeFilter(filter);
```

### 5. Identity and IdentityValidator Changes

Major refactoring in PR [#1158](https://github.com/eclipse-milo/milo/pull/1158) introduced the `Identity` interface.

**After (1.1.1):**
```java
IdentityValidator identityValidator = new IdentityValidator() {
    @Override
    public Object validateIdentity(
        Session session,
        Object tokenObject
    ) throws UaException {
        // Return Identity instance
        if (tokenObject instanceof UserNameIdentityToken token) {
            return new UsernameIdentity(
                token.getUserName(),
                token.getDecryptedPassword(session)
            );
        }
        return AnonymousIdentity.INSTANCE;
    }
};
```

### 6. Roles and Permissions

New support for Roles and Permissions added in PR [#1256](https://github.com/eclipse-milo/milo/pull/1256).

**After (1.1.1):**
```java
// Set roles on identity
UsernameIdentity identity = new UsernameIdentity(
    username,
    password,
    Set.of(
        WellKnownRole.AuthenticatedUser,
        new Role("CustomRole")
    )
);

// Check permissions on nodes
node.setRolePermissions(new RolePermission[] {
    new RolePermission(
        WellKnownRole.AuthenticatedUser.getNodeId(),
        PermissionType.Read.getValue()
    )
});
```

---

## Data Type Changes

### 1. BuiltinDataType Renamed to OpcUaDataType

In PR [#1377](https://github.com/eclipse-milo/milo/pull/1377), `BuiltinDataType` was renamed.

**Before (0.6.7):**
```java
BuiltinDataType dataType = BuiltinDataType.Int32;
```

**After (1.1.1):**
```java
OpcUaDataType dataType = OpcUaDataType.Int32;
```

### 2. Dynamic/Custom Data Types

Major improvements to dynamic data type support in PR [#1374](https://github.com/eclipse-milo/milo/pull/1374).

**After (1.1.1):**
```java
// Register custom structure type
StructureDefinition structDef = new StructureDefinition(
    /* ... */
);

DataTypeDefinition dataTypeDef = new DataTypeDefinition(
    structDef
);

// Server will automatically handle encoding/decoding
```

### 3. ExtensionObject Changes

`ExtensionObject` is now a sealed class in PR [#1387](https://github.com/eclipse-milo/milo/pull/1387).

**After (1.1.1):**
```java
// Pattern matching with sealed ExtensionObject
ExtensionObject xo = /* ... */;

Object decoded = switch (xo) {
    case ExtensionObject.Encoded encoded -> 
        codec.decode(encoded.getEncodedBody());
    case ExtensionObject.Decoded decoded -> 
        decoded.getDecodedBody();
};
```

### 4. Matrix Support for Multidimensional Arrays

New Matrix container for multidimensional arrays in PR [#1052](https://github.com/eclipse-milo/milo/pull/1052).

**After (1.1.1):**
```java
Matrix matrix = new Matrix(
    new int[] {2, 3}, // dimensions
    new Integer[] {1, 2, 3, 4, 5, 6} // elements
);

// Access elements
Integer value = (Integer) matrix.get(0, 1);
```

### 5. JSON Encoding Support

New JSON encoding implementation in PR [#1045](https://github.com/eclipse-milo/milo/pull/1045).

**After (1.1.1):**
```java
// Add dependency
<dependency>
    <groupId>org.eclipse.milo</groupId>
    <artifactId>milo-codec-json</artifactId>
    <version>1.1.1</version>
</dependency>

// Use JSON codec
JsonEncoder encoder = new JsonEncoder();
encoder.encodeMessage(message);
```

---

## Security and Certificates

### 1. Certificate Management Refactoring

Major refactoring to support Certificate Groups in PR [#1154](https://github.com/eclipse-milo/milo/pull/1154).

**After (1.1.1):**
```java
CertificateManager certificateManager = new DefaultCertificateManager(
    keyPair,
    certificate,
    certificateChain,
    applicationCertificateValidator,
    trustedCertificateValidator
);

// Support for multiple certificate groups
CertificateGroup defaultApplicationGroup = 
    certificateManager.getDefaultApplicationGroup();
```

### 2. CertificateValidator Interface

Common `CertificateValidator` interface introduced in PR [#1317](https://github.com/eclipse-milo/milo/pull/1317).

**After (1.1.1):**
```java
CertificateValidator validator = new CertificateValidator() {
    @Override
    public void validateCertificateChain(
        List<X509Certificate> certificateChain
    ) throws UaException {
        // Custom validation logic
    }
    
    @Override
    public void validateCertificateChain(
        List<X509Certificate> certificateChain,
        String applicationUri,
        String... validHostNames
    ) throws UaException {
        // Custom validation with URI and hostnames
    }
};
```

### 3. X.509 Extensions in CSR

Support for X.509 extensions in CSR generation added in PR [#1651](https://github.com/eclipse-milo/milo/pull/1651).

**After (1.1.1):**
```java
// Generate CSR with extensions
PKCS10CertificationRequest csr = CertificateBuilder
    .generateCSR(
        keyPair,
        subjectName,
        sanUri,
        sanDns
    );
```

### 4. Password Security

Passwords can now be supplied via `Supplier<byte[]>` for better security in PR [#1597](https://github.com/eclipse-milo/milo/pull/1597) and PR [#1617](https://github.com/eclipse-milo/milo/pull/1617).

**After (1.1.1):**
```java
// Password from supplier
client.getConfig().getIdentityProvider()
    .setPassword(() -> getPasswordFromSecureSource());

// X.509 identity from supplier
X509IdentityProvider identityProvider = 
    new X509IdentityProvider(
        () -> certificate,
        () -> privateKey
    );
```

---

## Dependencies

### Required Updates

**Before (0.6.7):**
- Java 8+
- Netty 4.1.x (older version)
- Guava (older version)

**After (1.1.1):**
- **Java 17+** (Required)
- Netty 4.1.105.Final
- Guava 32.1.3-jre
- JSpecify annotations
- Removed: Lombok (replaced with custom generated code)

### Maven Configuration

```xml
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
</properties>

<dependencies>
    <dependency>
        <groupId>org.eclipse.milo</groupId>
        <artifactId>milo-sdk-client</artifactId>
        <version>1.1.1</version>
    </dependency>
    <dependency>
        <groupId>org.eclipse.milo</groupId>
        <artifactId>milo-sdk-server</artifactId>
        <version>1.1.1</version>
    </dependency>
</dependencies>
```

### Testing Dependencies

Migration to JUnit 5 in PR [#1358](https://github.com/eclipse-milo/milo/pull/1358).

**Before (0.6.7):**
```xml
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.x</version>
    <scope>test</scope>
</dependency>
```

**After (1.1.1):**
```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.x</version>
    <scope>test</scope>
</dependency>
```

---

## Code Examples

### Complete Client Example

**0.6.7 Style:**
```java
import org.eclipse.milo.opcua.sdk.client.api.OpcUaClient;
import org.eclipse.milo.opcua.sdk.client.api.subscriptions.UaSubscription;
import org.eclipse.milo.opcua.stack.core.types.builtin.LocalizedText;
import org.eclipse.milo.opcua.stack.core.types.builtin.NodeId;

public class OldClientExample {
    public static void main(String[] args) throws Exception {
        OpcUaClient client = OpcUaClient.create(
            "opc.tcp://localhost:4840",
            endpoints -> endpoints.stream().findFirst(),
            configBuilder -> configBuilder
                .setApplicationName(LocalizedText.english("My Client"))
                .build()
        );
        
        client.connect().get();
        
        NodeId nodeId = new NodeId(2, "MyVariable");
        DataValue value = client.readValue(0, TimestampsToReturn.Both, nodeId).get();
        
        UaSubscription subscription = client.getSubscriptionManager()
            .createSubscription(1000.0).get();
            
        client.disconnect().get();
    }
}
```

**1.1.1 Style:**
```java
import org.eclipse.milo.opcua.sdk.client.OpcUaClient;
import org.eclipse.milo.opcua.sdk.client.subscriptions.ManagedSubscription;
import org.eclipse.milo.opcua.sdk.client.subscriptions.ManagedDataItem;
import org.eclipse.milo.opcua.stack.core.types.builtin.LocalizedText;
import org.eclipse.milo.opcua.stack.core.types.builtin.NodeId;

public class NewClientExample {
    public static void main(String[] args) throws Exception {
        OpcUaClient client = OpcUaClient.create(
            "opc.tcp://localhost:4840",
            builder -> builder
                .setApplicationName(LocalizedText.english("My Client"))
                .build()
        );
        
        client.connect().get();
        
        NodeId nodeId = new NodeId(2, "MyVariable");
        DataValue value = client.readValue(0, TimestampsToReturn.Both, nodeId).get();
        
        ManagedSubscription subscription = client.getSubscriptionManager()
            .createSubscription(1000.0).get();
            
        ManagedDataItem dataItem = subscription.createDataItem(nodeId);
        dataItem.addDataValueListener(v -> 
            System.out.println("Value: " + v.getValue().getValue())
        );
        
        client.disconnect().get();
    }
}
```

### Complete Server Example

**0.6.7 Style:**
```java
import org.eclipse.milo.opcua.sdk.server.OpcUaServer;
import org.eclipse.milo.opcua.sdk.server.api.config.OpcUaServerConfig;

public class OldServerExample {
    public static void main(String[] args) throws Exception {
        OpcUaServerConfig config = OpcUaServerConfig.builder()
            .setApplicationName(LocalizedText.english("My Server"))
            .setApplicationUri("urn:my:server")
            .build();
            
        OpcUaServer server = new OpcUaServer(config);
        server.startup().get();
    }
}
```

**1.1.1 Style:**
```java
import org.eclipse.milo.opcua.sdk.server.OpcUaServer;
import org.eclipse.milo.opcua.sdk.server.api.config.OpcUaServerConfig;
import org.eclipse.milo.opcua.sdk.server.identity.AnonymousIdentityValidator;

public class NewServerExample {
    public static void main(String[] args) throws Exception {
        OpcUaServerConfig config = OpcUaServerConfig.builder()
            .setApplicationName(LocalizedText.english("My Server"))
            .setApplicationUri("urn:my:server")
            .setIdentityValidator(new AnonymousIdentityValidator())
            .build();
            
        OpcUaServer server = new OpcUaServer(config);
        server.startup().get();
    }
}
```

---

## Troubleshooting

### Common Migration Issues

#### 1. "ClassNotFoundException" or "NoClassDefFoundError"

**Problem:** Classes moved or renamed during refactoring.

**Solution:** Update imports to remove "api" from package paths:
```java
// Old
import org.eclipse.milo.opcua.sdk.client.api.*;

// New
import org.eclipse.milo.opcua.sdk.client.*;
```

#### 2. "MethodNotFoundException" with Subscriptions

**Problem:** Subscription API completely redesigned.

**Solution:** Use `ManagedSubscription` and `ManagedDataItem` instead of `UaSubscription` and `UaMonitoredItem`.

#### 3. Java Version Incompatibility

**Problem:** Code won't compile with Java 8/11.

**Solution:** Upgrade to Java 17 or later:
```xml
<maven.compiler.source>17</maven.compiler.source>
<maven.compiler.target>17</maven.compiler.target>
```

#### 4. AddressSpace Async Methods Missing

**Problem:** `getNodeAsync()` and similar async methods no longer exist.

**Solution:** Use blocking methods or wrap in `CompletableFuture.supplyAsync()`:
```java
// Old
CompletableFuture<Optional<UaNode>> future = addressSpace.getNodeAsync(nodeId);

// New - direct blocking
Optional<UaNode> node = addressSpace.getNode(nodeId);

// New - if async needed
CompletableFuture<Optional<UaNode>> future = 
    CompletableFuture.supplyAsync(() -> addressSpace.getNode(nodeId));
```

#### 5. BuiltinDataType Compilation Errors

**Problem:** `BuiltinDataType` class not found.

**Solution:** Replace with `OpcUaDataType`:
```java
// Old
BuiltinDataType.Int32

// New
OpcUaDataType.Int32
```

#### 6. ExtensionObject Decode Issues

**Problem:** ExtensionObject handling changed to sealed class.

**Solution:** Use pattern matching:
```java
Object decoded = switch (extensionObject) {
    case ExtensionObject.Encoded encoded -> decode(encoded);
    case ExtensionObject.Decoded decoded -> decoded.getDecodedBody();
};
```

#### 7. AttributeContext Not Found

**Problem:** `AttributeContext` removed from API.

**Solution:** Replace with `AccessContext`:
```java
// Old
void method(AttributeContext context) { }

// New
void method(AccessContext context) { }
```

### Debugging Tips

1. **Enable Debug Logging:**
```java
System.setProperty("org.slf4j.simpleLogger.defaultLogLevel", "debug");
```

2. **Check DataType Encoding:**
```java
// Verify custom types are registered
DataTypeManager manager = client.getDynamicDataTypeManager();
Optional<DataTypeCodec> codec = manager.getCodec(typeId);
```

3. **Verify Security Configuration:**
```java
// Check certificate validity
X509Certificate cert = /* load cert */;
cert.checkValidity();
```

### Migration Checklist

- [ ] Upgrade to Java 17 or later
- [ ] Update Maven dependencies to 1.1.1
- [ ] Update JUnit dependencies to JUnit 5 (if applicable)
- [ ] Remove "api" from import statements
- [ ] Replace `BuiltinDataType` with `OpcUaDataType`
- [ ] Update client configuration builders
- [ ] Migrate subscription code to new API
- [ ] Replace `AttributeContext` with `AccessContext`
- [ ] Update `IdentityValidator` implementations
- [ ] Convert async AddressSpace calls to blocking
- [ ] Update certificate management code
- [ ] Test with new ExtensionObject sealed class
- [ ] Review and update custom AttributeFilters
- [ ] Verify all custom data types work correctly
- [ ] Test security configurations
- [ ] Run full test suite

---

## Additional Resources

- **Official Repository:** https://github.com/eclipse-milo/milo
- **Version 1.1.1 Release Notes:** https://github.com/eclipse-milo/milo/releases/tag/v1.1.1
- **Version 1.0.0 Release Notes:** https://github.com/eclipse-milo/milo/releases/tag/v1.0.0
- **Stack Overflow Tag:** [milo](http://stackoverflow.com/questions/tagged/milo)
- **Mailing List:** https://dev.eclipse.org/mailman/listinfo/milo-dev
- **Example Code:** https://github.com/eclipse-milo/milo/tree/main/milo-examples

---

## Summary

The migration from 0.6.7 to 1.1.1 is a significant undertaking that requires:

1. **Java 17 upgrade** - Non-negotiable requirement
2. **Import statement updates** - Remove "api" packages
3. **API refactoring** - Especially for client subscriptions and server address space
4. **Data type changes** - Rename and new implementations
5. **Security updates** - New certificate management patterns
6. **Testing** - Comprehensive testing of all OPC UA operations

Plan for adequate testing time and consider a phased migration approach if possible. The changes bring significant improvements in OPC UA 1.05 support, security, and data type handling that make the migration worthwhile.

---

**Document Version:** 1.0  
**Last Updated:** 2026-02-12  
**Milo Versions Covered:** 0.6.7 → 1.1.1
