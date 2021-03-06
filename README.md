# DigitalOcean Docker private registry module
[Terraform module](https://registry.terraform.io/modules/ximon18/docker-registry/digitalocean/) which creates a [private v2 Docker registry](https://docs.docker.com/registry/) on [DigitalOcean](https://digitalocean.com/).

## Disclaimers
I have not hardened the registry for production use - use at your own risk!

By using this module you accept the Lets Encrypt [Subscriber Agreement](https://letsencrypt.org/repository/)

This module does NOT encrypt the image storage and thus you are still dependent on the configured DigitalOcean access controls and on DigitalOcean data protection policy.

The rights of law enforcement agencies to access data stored by DigitalOcean may also depend on the region in which you deploy your registry.

I take no responsibility for the security of your data or the choices you make concerning how you store it.

## Features
* [Lets Encrypt](https://letsencrypt.org/) TLS certificate.
* Support for DigitalOcean Spaces as a storage backend for registry images.

## Usage
```
module "registry" {
    source              = "https://github.com/ximon18/terraform-digitalocean-docker-registry"
    ssh_key_fingerprint = "${digitalocean_ssh_key.somekey.fingerprint}"
    admin_password      = "SoMeP@SsW0rd"
    admin_email         = "your@email"
    fqdn                = "myregistry.some.domain.managed.by.digitalocean"
    region              = "digitalocean region e.g. ams3"
}
```

After deployment (and any DNS propagation delay) you can:

### View the registry catalog
Browse to https://\<fqdn\>/v2/_catalog
### Login to the registry
    docker login -u admin <fqdn>
### View the list of images stored in the registry
    docker image ls
### Push to the registry by first tagging then pushing
    docker tag <repo>/<image>:<tag> <fqdn>/<repo>/<image>/<tag>
    docker push <fqdn>/<repo>/<image>/<tag>
### Pull from the registry
    docker pull <repo>/<image>/<tag>
    docker pull <fqdn>/<repo>/<image>/<tag>
### Managing user access rights to the registry
Access is controlled by entries in an `/etc/registry/auth/htpasswd` file stored on the DigitalOcean Droplet. You can revoke access by deleting lines from the file. You can grant access to new users by executing a command like this on the Droplet when connected via SSH:

  	docker run --entrypoint htpasswd registry:2 -Bbn <username> "<password>" > /etc/registry/auth/htpasswd

And then because the registry [only reads the `htpasswd` file on startup](https://docs.docker.com/registry/configuration/#htpasswd) you'll need to restart the registry:

    docker restart registry

## Known issues
- At the time of writing the module deploys v2.5.2 of the Docker Registry as the newer version suffers from a backend panic with DigitalOcean Spaces. For more information see: https://github.com/docker/distribution/issues/2695.

- As this module uses the CertBot HTTP mechanism of proving domain ownership, it cannot be used to deploy the private registry on a domain name that is not accessible from the Lets Encrypt servers on the Internet.

## Further reading
See:
- Input variables and output "documentation" attributes.
- Comments in the module sources.
   
# Links
- https://docs.docker.com/registry/

# FAQ
Q: Why do you require control of a domain name?

A: To make it possible to obtain a TLS certificate from [Lets Encrypt](https://letsencrypt.org/). In my experience services using self-signed certificates, even when the self-signed nature of the certificate is okay, are a pain to use because tooling communicating with them either does not support self-signed certificates at all or requires configuration to work with them. If there is enough interest the module could be extended to support user-provided and/or self-signed TLS certificates instead which would remove the need for this requirement.

Q: Why don't you use NGINX like [DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-docker-registry-on-ubuntu-18-04) and even [Docker themselves](https://docs.docker.com/registry/deploying/#more-advanced-authentication) suggest?

A: I didn't want the additional complexity of a proxy in front of Docker, both for configuration, operation and when diagnosing issues, and Basic Authentication within TLS was good enough for my use case.

Q: Why don't you use the Docker out-of-the-box support for Lets Encrypt?

A: It's been broken since [issue 2545](https://github.com/docker/distribution/issues/2545).

Q: Why don't you use Caddy like [Paul Cody suggests](https://medium.com/@pcj/your-own-private-docker-repository-with-digitalocean-and-caddy-aug-26-2017-3e30859363ae)?

A: I didn't want the additional complexity (see above), and I experienced authentication issues when using Caddy, but it's possible that I had just made some other mistake in my testing at that point as others report no issues using Caddy in front of a Docker registry.

Q: Why don't you use the [DigitalOcean DNS plugin for CertBot](https://certbot-dns-digitalocean.readthedocs.io/en/stable/)?

A: I didn't want the additional complexity, and I didn't want to store my DigitalOcean API credentials on the registry Droplet. It could however be a way to enable deployment on private domains while still using Lets Encrypt.

Q: Why did you spend the time creating this? Didn't you see that XXX has already done this?

A: Partly as a learning exercise about Terraform. Despire some Googling I may have course missed something on the vast web so no I didn't see XXX and I would love to hear about any other solutions out there that I may have missed.

Q: How do I delete images?

A: Either by making [REST API](https://docs.docker.com/registry/spec/api/#deleting-an-image) calls manually or more easily by using the wonderful [Docker registry v2 command line client](https://github.com/genuinetools/reg) with command `reg rm -u <user> -p <pass> <registryfqdn>/<repo>/<image>:<tag>`. *Note:* You will need at least v0.0.1-beta of the Terraform template and have redeployed using `terraform apply`, otherwise your registry will not reject delete commands with HTTP 405.

Q: How do I cleanup space still used by deleted images?

A: SSH to the Docker Droplet as user root, then issue the following command:
```
docker exec -it registry bin/registry garbage-collect /etc/docker/registry//config.yml
```

END
