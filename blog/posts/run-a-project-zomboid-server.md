---
title: "Run a Project Zomboid dedicated server"
created_at: 2025-04-27T18:55:48Z
updated_at:
tags: ["devops", "hetzner", "opentofu", "ansible"]
cover: /assets/images/posts/esbuilds-build-api-is-pretty-cool/cover.jpg
custom:
    slug: esbuilds-build-api-is-pretty-cool
    summary: |
        Exploring the basics of esbuild's build API, to build SASS, CSS, and JS.
        As a sweet treat, how to package these all up with nix.
---

# {{ metadata.title }}

<div>
{% from "component/img.html" import img %}
</div>

<div>
{{ img(src=metadata.cover, alt="2 esbuild logo side by side in white and dark backgrounds") }}
</div>

<span class="post-metadata">
  {{ metadata.created_at|published_on(format="short") }}
</span>

<div>
{% from "component/tags.html" import tags %}
{{ tags(metadata.tags) }}
</div>

## Introduction

My friends and I have been talking about playing Project Zomboid for quite some
time. We recently got the 4-player bundle for those who didn't have it yet. For
convenience, the game comes with an option for one person to host the server
through Steam but it was rather tedious for us since basically the host had to
be there for the other one to play. Not to mention the game data is only
available to the host.

Fortunately for us, Project Zomboid allows you to run the server on a dedicated
machine. So for instance, in the event we need to move the game files around,
backups could be set up and made accessible for others to download.

So here's a quick guide to run a dedicated Project Zomboid server. For everyone,
or perhaps my friends would also try it out for something else. :)

## Spinning up a VPS

There are several cloud providers out there but since I don't have the luxury
to pay for AWS/GCP, I went with Hetzner instead.

You could just spin one up through Hetzner's UI but I'll need something to
automate the process of provisioning one, like [`opentofu`](https://opentofu.org/)!

```terraform
# pzserver.tf
terraform {
  required_version = "1.8.5"

  required_providers {
    hcloud = {
      source  = "hetznercloud/hcloud"
      version = "~> 1.50"
    }
  }
}
```

Here we're telling `opentofu` that we'd like to use Hetzner Cloud's provider.
This allows us to manage Hetzner resources.

We need a server from them, so let's add that.

```hcl
# pzserver.tf

# ...
resource "hcloud_server" "pzserver" {
  name        = "pzserver"
  server_type = "ccx13"
  datacenter  = "nbg1-dc3"
  ssh_keys    = data.hcloud_ssh_keys.kanto.ssh_keys[*].name
  image       = "ubuntu-24.04"

  public_net {
    ipv4_enabled = true
    ipv6_enabled = false
  }

  network {
    network_id = hcloud_network.network.id
    ip         = "10.0.1.2"
  }

  depends_on = [
    hcloud_network_subnet.subnet
  ]

  firewall_ids = [hcloud_firewall.server-firewall.id]
}
```

This creates an Ubuntu 24.04 server using Hetzner's `ccx13` which gets you a
dedicated 2-core AMD vCPU, and 8GB of memory which is just about right for the
number of players we intend to have. Somehow, it's cheaper than the 8GB memory
equivalent for a shared vCPU. Odd.

So if ever you want to spin up another server quickly, you no longer need to
manually do it through Hetzner's website. Now that we've provisioned the server,
let's set up the game server.

## Setting up the server

Project Zomboid has a fairly detailed wiki on how to run a game server, so if
you would like, you could check that out instead.

Since we're running it outside of our Steam account, we need to download and
install `steamcmd` to run the game's server.


```sh
sudo add-apt-repository multiverse; sudo dpkg --add-architecture i386; sudo apt update
sudo apt install steamcmd

sudo apt-get install software-properties-common -y
sudo apt-add-repository non-free
```

To run the game's server, we can avoid using the default `root` user since it
has too many dangerous permissions. So let's create one called `pzuser` solely
for running the game's server.

```sh
sudo adduser pzuser
```

Then create the destination for Project Zomboid's game server.

```sh
sudo mkdir /opt/pzserver
sudo chown pzuser:pzuser /opt/pzserver
```

Then we switch from `root` to `pzuser`, and login

```sh
sudo -u pzuser -i
```

Setting all of...that can be very repetitive and boring. Which is why tools like
`ansible` come in handy.
