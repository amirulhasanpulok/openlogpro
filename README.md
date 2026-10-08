# Openlog

Release channel for Openlog. This repository holds **releases only**, no source code. Each release has three files:

- `install.sh`: the installer
- `openlog-linux-amd64.tar.gz`: the release
- `openlog-linux-amd64.tar.gz.sha256`: its checksum

Install on a new Ubuntu 24.04 (x86_64) server, with the `install.sh` from the latest release:

    sudo bash install.sh --licence 'OL1....'

Update an installed server:

    sudo /usr/local/lib/openlog/update-log-server
