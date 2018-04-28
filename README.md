# `deploy-aws-docker-openvpn`

Deploys private OpenVPN servers quickly around the world via [Amazon AWS EC2](https://aws.amazon.com/ec2/).

Based on the [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn) docker container and inspired by
[brasey/deploy-docker-openvpn](https://github.com/brasey/deploy-docker-openvpn).

This project will deploy one (or more) `t2.nano` private VPN servers around the world to securely route your desktop,
laptop, router or mobile traffic. The tools in this package will help you configure a private, secure docker container
with an [OpenVPN](https://openvpn.net/) environment that you can easily deploy to any number of Amazon AWS
regions around the world with just a few lines of configuration, at any time.  Using
[Terraform](https://www.terraform.io/) you can easily setup or destroy your OpenVPN deployments in seconds.

## Goals

The goal is to create a OpenVPN certificate authority (CA) on your local machine via
[EasyRSA](https://github.com/OpenVPN/easy-rsa).  This CA will generate a server certificate for use in OpenVPN servers
that you can deploy via AWS, and generate client certificates for any machines you want to connect to those OpenVPN
servers.

We will then use [Terraform](https://www.terraform.io/) to deploy a docker container with the
[kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn) image to a minimal Amazon AWS EC2 server to
one or more AWS regions.  Terraform will also update [Amazon Route 53](https://aws.amazon.com/route53/) DNS entries
pointing to your deployed server(s).

You can then use the `.ovpn` profiles generated by the CA on your local machine to connect to your OpenVPN servers
across the globe!

# Steps

## 1. Install Docker

First, you will need have [Docker](https://www.docker.com/) installed on your local machine:

* [Docker for Mac](https://docs.docker.com/docker-for-mac/)
* [Docker for Windows](https://docs.docker.com/docker-for-windows/)

## 2. Local OpenVPN CA Configuration

Next, we'll create a Docker volume on the local machine to host the CA files.

Then, we will use the [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn) docker image
to generate a OpenVPN certificate authority (CA) with a CA root key.  This CA will be responsible for generating
certificates for all of your servers and clients.  You won't deploy this container directly to the outside world --
it will be solely responsible for generating server certificates for your OpenVPN servers and client certificates
for each of your clients, with the CA root key. These files will live in the Docker volume we created.

This is a modified process of the [kylemanna/docker-openvpn Quick Start](https://github.com/kylemanna/docker-openvpn#quick-start)
guide where we use the local machine to generate the CA root key and
[only put the CA certificate](https://github.com/kylemanna/docker-openvpn/blob/master/docs/paranoid.md)
on the deployed OpenVPN servers.  The files required for the OpenVPN servers will be generated into
`files/openvpn-server.tar.gz`.

1. Configure a few environment variables.

    Replace `vpn.myhost.com` with the [Amazon Route 53](https://aws.amazon.com/route53/) DNS entry
    you want your clients to connect to.

    Linux/Mac:

    ```shell
    # Set this to the VPN server name
    OVPN_SERVER="vpn.myhost.com"
    OVPN_DATA="ovpn-data"
    ```

    Windows:

    ```batch
    REM Set this to the VPN server name
    set OVPN_SERVER=vpn.myhost.com
    set OVPN_DATA=ovpn-data
    ```

2. Create a Docker volume

    Linux/Mac:

    ```shell
    docker volume create --name $OVPN_DATA
    ```

    Windows:

    ```batch
    docker volume create --name %OVPN_DATA%
    ```

2. Next, we'll generate the OpenVPN configuration into the `$OVPN_DATA` voume:

    Linux/Mac:

    ```shell
    # Generates OpenVPN configs
    docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://$OVPN_SERVER
    ```

    Windows:

    ```batch
    REM Generates OpenVPN configs
    docker run -v %OVPN_DATA%:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://%OVPN_SERVER%
    ```

3. We'll use [EasyRSA](https://github.com/OpenVPN/easy-rsa) to build the CA root key, certificate and other
    files used by the server.

    Linux/Mac:

    ```shell
    # Initialize the EasyRSA PKI
    docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
    ```

    Windows:

    ```batch
    REM Initialize the EasyRSA PKI
    docker run -v %OVPN_DATA%:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
    ```

4. Finally, we'll call `ovpn_copy_server_files` which will copy out the minimal files required for the OpenVPN
    server into `files/openvpn-server.tar.gz`.

    Linux/Mac:

    ```shell
    # Generates the minimal OpenVPN files necessary for clients to connect to into /etc/openvpn/server
    docker run --net=none --rm -t -i -v $OVPN_DATA:/etc/openvpn kylemanna/openvpn ovpn_copy_server_files

    # Package the files up into an archive
    docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn tar -cvz -C /etc/openvpn/server . > files/openvpn-server.tar.gz
    ```

    Windows:

    ```batch
    REM Generates the minimal OpenVPN files necessary for clients to connect to into /etc/openvpn/server
    docker run --net=none --rm -t -i -v %OVPN_DATA%:/etc/openvpn kylemanna/openvpn ovpn_copy_server_files

    REM Package the files up into an archive
    docker run -v %OVPN_DATA%:/etc/openvpn --rm kylemanna/openvpn tar -cvz -C /etc/openvpn/server . > files/openvpn-server.tar.gz
    ```

Now we're all set to generate client certificates and deploy our servers.

## 3. Client Configuration

For each client you want to allow to connect to your OpenVPN instances, we will use
[EasyRSA](https://github.com/OpenVPN/easy-rsa) to generate client certificates and the `ovpn_getclient` script
to build `.ovpn` files that can be used by OpenVPN client software.

The client certificates will end up in `$OVPN_DATA/pki/issued/` and the `.ovpn` profiles will be written out to the local
machine.

Linux/Mac:

```shell
CLIENTNAME=myclient

docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full $CLIENTNAME nopass
docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient $CLIENTNAME > $CLIENTNAME.ovpn
```

Windows:

```batch
set CLIENTNAME=myclient

docker run -v %OVPN_DATA%:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full %CLIENTNAME% nopass
docker run -v %OVPN_DATA%:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient %CLIENTNAME% > %CLIENTNAME%.ovpn
```

Note if you're going to be deploying multiple VPN instances you'll want to modify the `.ovpn` to edit the `remote`
DNS entry.

## 4. Deployment

Once the server has been configured and you have generated keys for your clients, you are ready to deploy OpenVPN
servers to AWS.

We will be using [Terraform](https://www.terraform.io/) to easy prepare and deploy OpenVPN servers across the globe.
They will be initialized using the certificates generated via `ovpn_copy_server_files` and in the
`files/openvpn-server.tar.gz` package.

Terraform uses configuration files to know how to configure each OpenVPN deployment.  The base configuration is in
`terraform.tfvars`, and each deployment has its own `config-[deployment].tfvars` file.  For example, you could deploy
OpenVPN servers to multiple AWS regions.

First, we'll need to configure Amazon AWS:

1. Create an [IAM](https://console.aws.amazon.com/iam/home) user that will be responsible for deploying the EC2 instances
    and for modifying the Route 53 domain entries.

    When you create the IAM user, you will be given a AWS Access Key and a AWS Secret Key that you will use later.

    You will need to attach some permissions to this user.  You may be able to attach finer-grained policies, but these
    should be sufficient:

    * `AmazonEC2FullAccess`
    * `AmazonRoute53DomainsFullAccess`
    * `AmazonRoute53ReadOnlyAccess`
    * `AmazonSSMFullAccess`

2. Create (or use an existing) [Amazon Route 53](https://console.aws.amazon.com/route53/home) Hosted Zone (root domain)
    that will host the DNS record you want updated when you deploy a server.

    Note the Hosted Zone ID.

3. Generate (or use an existing) Private Key pair that you will use to connect (via SSH) to EC2 instances.

    The easiest way to do this is via the [EC2 console](https://eu-central-1.console.aws.amazon.com/ec2/v2/home).  Click
    on _Key Pairs_ and _Create Key Pair_.  You wil lbe given a `.pem` file with your private keys.

    If you already have a Private Key, you will need to make sure it's listed in the EC2 _Key Pairs_ console for every
    zone that you want to deploy OpenVPN servers to.

    If you're deploying to more than one AWS zone, you will use _Import Key Pair_ to import your `.pem` to other zones.

Next, we'll need to edit a couple configuration files.

1. `terraform.tfvars`

    You can use the `terraform.tfvars.sample` file to get you started.  Copy it to `terraform.tfvars`:

    ```
    aws_access_key = ""
    aws_secret_key = ""
    host_route_53_zone_id = ""
    instance_type = "t2.nano"
    ssh_key = "mykey.pem"
    ssh_key_name = ""
    ssh_user_name = "ec2-user"
    ```

    * `aws_access_key` is your AWS Access Key
    * `aws_secret_key` is your AWS Secret Key
    * `host_route_53_zone_id` is the Hosted Zone ID you will be updating for `$OVPN_SERVER`
    * `instance_type` can be the [AWS EC2 instance type](https://aws.amazon.com/ec2/instance-types/) you want to use.  At
        the time of writing, `t2.nano` is the cheapest instance type.
    * `ssh_key` is the location of the `.pem` file you created
    * `ssh_key_name` is the name of _Key Pair Name_ in EC2
    * `ssh_user_name` is the user name used to connect to your EC2 instance.  `ec2-user` should probably be used.

2. Create `config-[deployment].tfvars` for each deployment (e.g. zone) you want to deploy to.

    You can use the `config-main.tfvars.sample` file to get you started.  For example, copy it to `config-us-east.tfvars`:

    ```
    host_name = "vpn.myhost.com"
    region = "us-east-2"
    base_image = "ami-31c7f654"
    ```

    * `host_name` is the DNS name of this deployment
    * `region` is the AWS region you want to deploy to
    * `base_image` is your AMI you want to use _for that region_.  This package assumes
        [Amazon Linux 2](https://aws.amazon.com/amazon-linux-2/).

3. Initialize Terraform:

    ```
    terraform init
    ```

4. Create Terraform Workspaces for each deployment:

    For example, if you want to deploy to `us-east`, `us-west` and `de`:

    ```
    terraform workspace new us-east
    terraform workspace new us-west
    terraform workspace new de
    ```

Now you're ready to deploy!  The Terraform scripts will do the following:

* Configures a VPC, Internet Gateway, Routes and Subnet for this region (required if you're using `t2.nano`)
* Creates a Security Group (firewall), open to incoming SSH (22) and OpenVPN (1194)
* Launches an EC2 instance
    * Once it's ready, it uploads the files in `files/`:
        * `openvpn-server.tar.gz` package that has the OpenVPN server certificate
        * `docker-openvpn@data.service` to ensure the OpenVPN runs after reboots
    * Runs `scripts/configure_node.sh` to:
        * Update the packages on the machine
        * Install Docker
        * Install `docker-openvpn` as a service
        * Create a docker volume for `/etc/openvpn` and extract `openvpn-server.tar.gz` into it
        * Start the `docker-openvpn` service

To deploy to a region, you simply need to switch to that workspace, then run `terraform apply` with
the `-var-file config-[deployment].tfvars` set:

```
terraform workspace select us-east
terraform plan -var-file config-us-east.tfvars
terraform apply -var-file config-us-east.tfvars

terraform workspace select us-west
terraform plan -var-file config-us-west.tfvars
terraform apply -var-file config-us-west.tfvars

terraform workspace select de
terraform plan -var-file config-de.tfvars
terraform apply -var-file config-de.tfvars
```

You can destroy (stop) a deployment via:

```
terraform workspace select us-east
terraform destroy -var-file config-us-east.tfvars
```