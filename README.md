# BOMIST Enterprise Utils

## Infrastructure

We recommend using [Ansible](https://docs.ansible.com/?extIdCarryOver=true&sc_cid=701f2000001OH7YAAW) to automate the installation of **CouchDB** on a remote server as well as enabling **HTTPS** with automatic SSL certificate renewal.

If you don't have it yet, start by [installing Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-and-upgrading-ansible-with-pipx) on your local machine. On Windows, make sure to install it and use it under WSL (Windows Subsystem for Linux).

This guide assumes your server runs **Ubuntu 22.04**.

### Adding an SSH key to your server

In order for Ansible to be able to communicate with your server, your host machine (where you run Ansible from) should have an SSH key and the corresponding public part should be added to your server.

To generate a new SSH key, in case you don't have one yet (replace `your_email@example.com` with your actual email):

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

You can now check your public key by running:

```
cat ~/.ssh/id_ed25519.pub
```

Notice the file ends with `.pub` indicating it's the _public_ part of your SSH key. The text you get here is what you should add to your server. The way you add the SSH public key to your server will depend on you cloud provider (or rather, its user interface) but it should be a straightforward step.

To make sure you can SSH into your server, and so will Ansible, you can run this (replace `<server_ip>` with your actual server's IP):

```
ssh root@<server_ip>
```

_Side note_: creating a server and copying your local SSH public key into it could also be automated by using a tool such as [Terraform](https://www.terraform.io). However, since this is probably something you won't need to do often, we leave it outside the scope of this guide.

### Defining variables

Make sure you are at the `infra/ansible` directory of this repository.

```
$ cd infra/ansible
```

Create a `hosts` file that lists the server the CouchDB should run on. For example:

```
[couchdb]
xxx.xxx.xx.xxx ansible_user=root
```

Edit the `couchdb/vault.yml` file and define these variables:

```
couchdb_user=
couchdb_password=
couchdb_jwt_secret_b64=
```

Note that `couchd_jwt_secret_b64` should represent the JSON Web Token secret encoded in `base64` format. The non-encoded text should be the same as defined on the environment variable `BOMIST_ENTERPRISE_JWT_SECRET` when running the Enterprise API (see [here](https://enterprise.bomist.com/configuration#environment-variables)).

Then:

```
ansible-vault encrypt couchdb/vault.yml
```

This will encrypt the variables you defined just above so the `couchdb/vault.yml` file can be safely shared.

### Installing CouchDB

This Ansible playbook installs CouchDB and sets it up with the configuration defined at `couchdb/templates/local.ini.j2` by using the variables defined above.

```
ansible-playbook -i hosts couchdb/main.yml --ask-vault-pass
```

### Enabling HTTPS

This requires having a public domain. Once you have one, make sure to create a DNS `A record` so the domain, or a sub-domain, points to your server's IP.

[Caddy](https://caddyserver.com) is used here for [automatic HTTPS](https://caddyserver.com/docs/automatic-https) and as a reverse proxy redirecting traffic to CouchDB.
This Ansible playbook installs Caddy and sets it up as a reverse proxy.

Replace `yourdomain.com` with your actual domain (or a sub-domain, for example `couch.yourdomain.com`).

```
ansible-playbook -i hosts caddy/main.yml --extra-vars "domain=yourdomain.com port=5984"
```

After this, the database should be available at `https://yourdomain.com`.

This ansible playbook can also be used to enable HTTPS on any remote server, not necessarily for the CouchDB (for example, the server the Enterprise API is running on). Just make sure you pass the appropriate `domain` and `port`.

### Safety considerations

If you run CouchDB on the cloud, make sure HTTPS is enabled and consider having a firewall allowing traffic only on ports `80` and `443` so only HTTP and HTTPS traffic is allowed. Caddy, mentioned above, automatically redirects HTTP traffic to HTTPS.

### Fauxton: the CouchDB dashboard

CouchDB comes with a web-based interface called [Fauxton](https://couchdb.apache.org/fauxton-visual-guide/index.html#using-fauxton) that allows you to easily interact with the database, namely creating, updating, deleting and viewing documents.

Once CouchDB is running, Fauxton becomes available at `<couchdb_url>/_utils`. For example, considering you have CouchDB running on https://couch.yourdomain.com, Fauxton would be available at https://couch.yourdomain.com/_utils.
