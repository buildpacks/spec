# Buildpack API v3 - Ideas

## Support for Windows & Running the Buildpack from Source

UNIX, valid executable names in order: /bin/build, /bin/build.sh

Windows, valid executable names, in order: /bin/build.exe, /bin/build.ps1, /bin/build.bat

The lifecycle MAY accept a user-provided plan.toml as an alternative to detection. 
