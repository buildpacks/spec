# Buildpack API v3 - Specification

This specification defines interactions between a platform, a lifecycle, a number of buildpacks, and an application
1. For the purpose of transforming that application into an OCI image and
2. For the purpose of developing or executing automated tests on that application.

A **buildpack** is software that partially or completely transforms application source code into runnable artifacts.

A **lifecycle** is software that orchestrates buildpacks and transforms the resulting artifacts into an OCI image.

A **platform** is software that orchestrates a lifecycle to make buildpack functionality available to end-users such as application developers.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" are to be interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119).

The key words "unspecified", "undefined", and "implementation-defined" are to be interpreted as described in the [rationale for the C99 standard](http://www.open-std.org/jtc1/sc22/wg14/www/C99RationaleV5.10.pdf#page=18).

An implementation is not compliant if it fails to satisfy one or more of the MUST, MUST NOT, REQUIRED, SHALL, or SHALL NOT requirements for the protocols it implements.
An implementation is compliant if it satisfies all the MUST, MUST NOT, REQUIRED, SHALL, and SHALL NOT requirements for the protocols it implements.

## Sections

- [Buildpack Interface Specification](buildpack.md)
- [Platform Interface Specification](platform.md)
- [Distribution Specification](distribution.md)
