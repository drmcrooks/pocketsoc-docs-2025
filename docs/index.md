# PocketSOC Documentation

Welcome to the official documentation for **PocketSOC**. PocketSOC is a SOC training platform that leverages open-source tools such as Docker, Portainer, Zeek, MISP, OpenSearch, Logstash to deploy Security Operation Center (SOC) training environment, referred to as PocketSOC.

The tools and technologies mentioned - Docker, Portainer, Zeek, MISP, OpenSearch, Logstash are commonly used in cybersecurity and SOC environments.

All of these components run as Docker Containers on a VM which provides one instance of PocketSOC for a trainee. Attendees interact with their PocketSOC instance using Portainer, which runs on each VM and provides a web UI to interact with each of the containers running on a host.

A head node, running on one VM, is used as the main or controller node of the PocketSOC. It uses Ansible to run these Docker containers on X Managed nodes. 

## Overview

This documentation will guide you through:

- Setting up the environment.
- Installation and configuration.
- Deployment for admin.
- Access for the trainee.


## Commands

* `mkdocs new [dir-name]` - Create a new project.
* `mkdocs serve` - Start the live-reloading docs server.
* `mkdocs build` - Build the documentation site.
* `mkdocs -h` - Print help message and exit.

## Project layout

    mkdocs.yml    # The configuration file.
    docs/
        index.md  # The documentation homepage.
        Admin/
            deployment.md
            design.md 
        Trainees/
            access.md
            excercise_one.md
            excercise_two.md
            excercise_three.md
            excercise_four.md
            excercise_five.md
            excercise_six.md
            images/
            ...   # All the images      
        ...       # Other pages, images and files.


## About 

As part of improving STFC’s cybersecurity posture, the SOC is being set up to address network traffic monitoring capability shortfalls. Since this project was spawned from the Tier-1, and the Tier-1 is part of the Worldwide LHC Computing Grid (WLCG), this is a key forum for us through which knowledge on security in research computing is shared. Security is often a part of this - in fact there is a specific Thematic CERN School of Computing (tCSC) dedicated to security. At tCSC there is a section on SOCs, and PocketSOC is the tool used to give trainees a dedicated training environment.
