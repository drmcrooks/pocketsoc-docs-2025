# PocketSOC Documentation

Welcome to the official documentation for **PocketSOC**. PocketSOC is a SOC training platform that leverages open-source tools such as Docker, Portainer, Zeek, MISP, OpenSearch, Logstash to deploy Security Operation Center (SOC) training environment, referred to as PocketSOC.

The tools and technologies mentioned - Docker, Portainer, Zeek, MISP, OpenSearch, Logstash are commonly used in cybersecurity and SOC environments.

All of these components run as Docker Containers on a VM which provides one instance of PocketSOC for a trainee. Attendees interact with their PocketSOC instance using Portainer, which runs on each VM and provides a web UI to interact with each of the containers running on a host.

A head node, running on one VM, is used as the main or controller node of the PocketSOC. It uses Ansible to run these Docker containers on X Managed nodes. 

## Overview

This documentation will guide you through:

- Setting up the environment.
- Installing and configuring each component.
- Troubleshooting common issues.
- Contributing to the project.


For full documentation visit [mkdocs.org](https://www.mkdocs.org).

# Commands

* `mkdocs new [dir-name]` - Create a new project.
* `mkdocs serve` - Start the live-reloading docs server.
* `mkdocs build` - Build the documentation site.
* `mkdocs -h` - Print help message and exit.

# Project layout

    mkdocs.yml    # The configuration file.
    docs/
        index.md  # The documentation homepage.
        ...       # Other markdown pages, images and other files.
