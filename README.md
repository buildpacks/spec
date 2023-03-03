# Cloud Native Buildpacks Specification v3

This specification defines interactions between a platform, a lifecycle, a number of buildpacks, and an application
1. For the purpose of transforming that application into an OCI image and
2. For the purpose of developing or executing automated tests on that application.

A **buildpack** is software that partially or completely transforms application source code into runnable artifacts.

A **lifecycle** is software that orchestrates buildpacks and transforms the resulting artifacts into an OCI image.

A **platform** is software that orchestrates a lifecycle to make buildpack functionality available to end-users such as application developers.

## Notational Conventions

### Key Words
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" are to be interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119).

The key words "unspecified", "undefined", and "implementation-defined" are to be interpreted as described in the [rationale for the C99 standard](http://www.open-std.org/jtc1/sc22/wg14/www/C99RationaleV5.10.pdf#page=18).

An implementation is not compliant if it fails to satisfy one or more of the MUST, MUST NOT, REQUIRED, SHALL, or SHALL NOT requirements for the protocols it implements.
An implementation is compliant if it satisfies all the MUST, MUST NOT, REQUIRED, SHALL, and SHALL NOT requirements for the protocols it implements.

### Operating System Conventions

When a word or bullet point is prefixed with a <a name="linux-only">†</a>, it SHALL be assumed to apply only to Linux stacks.

When a word or bullet point is prefixed with a <a name="windows-only">‡</a>, it SHALL be assumed to apply only to Windows stacks.


When the specification denotes a "shell", Linux stacks MUST use the Bourne Again Shell (`bash`) version 3 or greater and Windows stacks MUST use Command Prompt (`cmd.exe`) unless otherwise specified.

#### Interpreting Paths for Windows

When the specification denotes a filesystem path using POSIX path notation (e.g. `/cnb/lifecycle`), this notation SHALL be interpreted to represent a path where all POSIX file path separators are replaced with the Windows filepath separator (`\`) and absolute paths are assumed to be rooted in the default drive (e.g. `C:\cnb\lifecycle`).

When the specification refers to an executable file with POSIX path notation (e.g. `/cnb/buildpacks/bp-a/1.2.3/bin/detect`), this notation SHALL be interpreted to represent one of two possible files: one with the suffix `.exe` (e.g. `C:\cnb\buildpacks\bp-a\1.2.3\bin\detect.exe`) or with the suffix `.bat` (e.g. `C:\cnb\buildpacks\bp-a\1.2.3\bin\detect.bat`).

When the specification refers to a path in the context of an OCI layer tar (e.g. `/cnb/buildpacks/bp-a/1.2.3/`), this path SHALL be interpreted to be prefixed with `Files` (e.g. `Files/cnb/buildpacks/bp-a/1.2.3/`). Note: path separators in OCI layer tar headers MUST be `/` regardless of operating system.

## Sections

- [Buildpack Interface Specification](buildpack.md)
- [Platform Interface Specification](platform.md)
- [Distribution Specification](distribution.md)
- [Bindings Extension Specification](extensions/bindings.md)
- [Buildpack Registry Extension Specification](extensions/buildpack-registry.md)
- [Project Descriptor Extension Specification](extensions/project-descriptor.md)

## API Versions

These documents currently specify:

- Buildpack API: `0.9`
- Distribution API: `0.3`
- Platform API: `0.12`
