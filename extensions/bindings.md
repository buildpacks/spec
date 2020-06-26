# Bindings
Bindings are exposed inside of a container during all three phases of the lifecycle; detect, build, and launch.  The contents of bindings MUST NOT be part of the image created after the detect and build phases.

## Detect and Build Phases
During the detect and build phases, the lifecycle MUST provide any bindings as files in `<platform>/bindings/<binding-name>` with directory names matching the name of the binding.  Binding names MUST match `[a-z0-9\-\.]{1,253}`.

### Metadata
Within each binding directory the lifecycle MUST provide a `metadata` directory containing `kind`, `provider`, and `tags` files.  The value of the `kind` file MUST contain an abstract classification of the binding.  The value of the `provider` file MUST identify the provider of this binding.  The value of the `tags` file MUST contain a newline delimited collection of tags attached to the binding.

In addition to the required files, the `metadata` directory MAY contain additional metadata about the binding with file names and contents matching the metadata names and contents.

The collection of files within the directory MAY change between detect and build phase pairs.  The collection of files within the directory MUST NOT change during the detect and build phase pair.

The contents of the files MAY change between detect and build phase pairs.  The contents of the files MUST NOT change during the detect and build phase pair.

### Secret
Within each binding directory the lifecycle MAY provide a `secret` directory containing the secret associated with the binding with filenames matching the secret key names.

During the detect and build phases, if the `secret` directory exists, the contents of the files MAY be one of the following:

* the values of the secret keys
* an indirect pointer to another system for value resolution

If the `secret` directory exists, the collection of files within the directory MAY change between detect and build phase pairs.  The collection of files within the directory MUST NOT change during the detect and build phase pair.

If the `secret` directory exists, the contents of the files MAY change between detect and build phase pairs.  The contents of the files MUST NOT change during the detect and build phase pair.


### Example Directory Structure
```plain
<platform>
└── bindings
    ├── primary-db
    │   └── metadata
    │       ├── connection-count
    │       ├── kind
    │       ├── provider
    │       └── tags
    └── secondary-db
        ├── metadata
        │   ├── connection-count
        │   ├── kind
        │   ├── provider
        │   └── tags
        └── secret
            ├── endpoint
            ├── password
            └── username
```

## Launch Phase
During the launch phase, the lifecycle MUST provide any bindings as files in `$CNB_BINDINGS/<binding-name>` with directory names matching the name of the binding.  Binding names MUST match `[a-z0-9\-\.]{1,253}`.  The `CNB_BINDINGS` environment variable MUST be declared and can point to any valid filesystem location.

### Metadata
Within each binding directory the lifecycle MUST provide a `metadata` directory containing `kind`, `provider`, and `tags` files.  The value of the `kind` file MUST contain an abstract classification of the binding.  The value of the `provider` file MUST identify the provider of this binding.  The value of the `tags` file MUST contain a newline delimited collection of tags attached to the binding.

In addition to the required files, the `metadata` directory MAY contain additional metadata about the binding with file names and contents matching the metadata names and contents.

The collection of files within the directory MAY change between launches.  The collection of files within the directory MUST NOT change during the launch phase.

The contents of the files MAY change between launches.  The contents of the files MAY change during the launch phase.

### Secret
Within each binding directory the lifecycle MUST provide a `secret` directory containing the secret associated with the binding with filenames matching the secret key names.

During the launch phase, the contents of the files MAY be one of the following:

* the values of the secret keys
* an indirect pointer to another system for value resolution

The collection of files within the directory MAY change between launches.  The collection of files within the directory MUST NOT change during the launch phase.

The contents of the files MAY change between launches.  The contents of the files MAY change during the launch phase.

### Example Directory Structure
```plain
custom-bindings-location
├── primary-db
│   ├── metadata
│   │   ├── connection-count
│   │   ├── kind
│   │   ├── provider
│   │   └── tags
│   └── secret
│       ├── endpoint
│       ├── password
│       └── username
└── secondary-db
    ├── metadata
    │   ├── connection-count
    │   ├── kind
    │   ├── provider
    │   └── tags
    └── secret
        ├── endpoint
        ├── password
        └── username
```
