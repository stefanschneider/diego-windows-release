## Installing

This document will go over the steps required to setup a Windows cell in a
working CF/Diego deployment

## Requirements

- working CF/Diego deployment
- Windows Server 2012R2 VM (we recommend r3.xlarge, see https://github.com/cloudfoundry-incubator/diego-release/commit/c9331bc1b1000bd135cb99a025a3680d1a12ac87)
  - Recommended Windows ISO SHA1: B6F063436056510357CB19CB77DB781ED9C11DF3

## Retrieve the MSI

### Building from source

See [BUILDING.md](BUILDING.md) for further instructions.

### Download a prebuilt MSI

You can download our latest Garden MSIs from
[here](https://github.com/pivotal-cf/garden-windows-release/releases/latest)
and the latest DiegoWindows MSIs from
[here](https://github.com/pivotal-cf/diego-windows-release/releases/latest].

## Setup the Windows cell

### CloudFormation

There is a CloudFormation template in the root of the
[garden-windows-release](https://github.com/cloudfoundry-incubator/diego-windows-release/)
repository. This template can be uploaded to [Cloud
Formation](https://console.aws.amazon.com/cloudformation/home) for automatic
setup of a Windows cell.

The Cloud Formation wizard will ask for a number of parameters.

1. SecurityGroup: Security group ID to use for the Windows cells
1. BoshUserName: Username for BOSH director
1. BoshPassword: Pasword for BOSH director (Make sure your password meets [Windows complexity requirements](https://technet.microsoft.com/en-us/library/Cc786468(v=WS.10).aspx))
1. BoshHost: Bosh director host
1. ContainerizerPassword: Pasword for containerizer user e.g. password123!
1. CellName: The name for your cell
1. VPCID: the id of the vpc in which the cell and the subnet will be created
1. NATZ: the instance id of the `NATZ` box
1. SubnetCIDR: the ip range of the Windows cell subnet, e.g. `10.10.100.0/24`

The Cloud Formation template will configure the Windows cell for the
appropriate availability zone based on the provided security group, install the
MSI and register itself with Diego. The Cloud Formation template will only
succeed if all services are up and running after installation. To debug a
failed install, set "Rollback on failure" to "No" under advanced options.

### Manual Setup

1. Download the `setup.ps1` from
our [latest release](https://github.com/pivotal-cf/diego-windows-release/releases/latest).
From inside File explorer right click on the file and click `Run with powershell`.
The script will enable the required Windows features
, configure the DNS settings, and configure the firewall to the way that the cell needs.

## Install the MSIs

### Option 1: Using the [Install Script Generator](https://github.com/cloudfoundry-incubator/greenhouse-install-script-generator)

1. Download the `generate.exe` from the same release. Run it with the following argument template:

```
generate.exe -outputDir=[the directory where the script will output its files]
             -windowsUsername=[the username of an administrator user for Containerizer to run as]
             -windowsPassword=[the password for the same user] 
             -boshUrl=[the URL for your BOSH director, with credentials]
```

For example:

```
generate.exe -outputDir=C:\diego-install-dir -windowsUsername=Administrator -windowsPassword=MyPass123! -boshUrl=https://10.10.0.54:25555
```

1. Download `DiegoWindows.msi` and `GardenWindows.msi` to the output directory
you specified to the generate command. The filenames being correct is
important, since the script assumes these will be the MSI file names.

1. Run the `install.bat` script in the output directory. This will install
both of the MSIs with all of the arguments they require.

### Option 2: Manually

The following instructions assume that the MSIs were downloaded to `c:\temp`

```
msiexec /norestart /i c:\temp\GardenWindows.msi ^
          ADMIN_USERNAME=[Username with admin privileges] ^
          ADMIN_PASSWORD=[Previous user password] ^
          SYSLOG_HOST_IP=[(optional) Syslog host IP to send logs to] ^
          SYSLOG_PORT=[(optional) Syslog port to send logs to]

msiexec /norestart /i c:\temp\DiegoWindows.msi ^
          BBS_CA_FILE=[(optional) path to the BBS CA certificate] ^
          BBS_CLIENT_CERT_FILE=[(optional) path to the BBS client certificate] ^
          BBS_CLIENT_KEY_FILE=[(optional) path to the BBS client key] ^
          CONSUL_IPS=[Comma-separated IP addresses of consul agents from BOSH deploy of CF] ^
          CONSUL_ENCRYPT_FILE=[path to the consul encryption key] ^
          CONSUL_CA_FILE=[path to the consul CA certificate] ^
          CONSUL_AGENT_CERT_FILE=[path to the consul agent certificate] ^
          CONSUL_AGENT_KEY_FILE=[path to the consul agent key] ^
          CF_ETCD_CLUSTER=[URI of your Elastic Runtime cf etcd cluster from BOSH deploy of cf] ^
          STACK=[CF stack, eg. windows2012R2] ^
          REDUNDANCY_ZONE=windows ^
          LOGGREGATOR_SHARED_SECRET=[loggregator secret from your BOSH deploy of cf] ^
          EXTERNAL_IP=[(optional) External IP of box] ^
          MACHINE_NAME=[(optional) This machine's name (must be unique across your cluster)] ^
          SYSLOG_HOST_IP=[(optional) Syslog host IP to send logs to] ^
          SYSLOG_PORT=[(optional) Syslog port to send logs to]
```

An example would be:

```
msiexec /norestart /i c:\temp\GardenWindows.msi ^
          ADMIN_USERNAME=Administrator ^
          ADMIN_PASSWORD=secret^%password ^
          SYSLOG_HOST_IP=syslog-server.example.com ^
          SYSLOG_PORT=514

msiexec /norestart /i c:\temp\DiegoWindows.msi ^
          ADMIN_USERNAME=Administrator ^
          ADMIN_PASSWORD=secret^%password ^
          BBS_CA_FILE=c:\temp\bbs_ca.crt ^
          BBS_CLIENT_CERT_FILE=c:\temp\bbs_client.crt ^
          BBS_CLIENT_KEY_FILE=c:\temp\bbs_client.key ^
          CONSUL_IPS=10.10.5.11,10.10.6.11,10.10.7.11 ^
          CONSUL_ENCRYPT_FILE=c:\temp\consul_encrypt.key ^
          CONSUL_CA_FILE=c:\temp\consul_ca.crt ^
          CONSUL_AGENT_CERT_FILE=c:\temp\consul_agent.crt ^
          CONSUL_AGENT_KEY_FILE=c:\temp\consul_agent.key ^
          CF_ETCD_CLUSTER=http://10.244.0.42:4001 ^
          STACK=windows2012R2 ^
          REDUNDANCY_ZONE=windows ^
          LOGGREGATOR_SHARED_SECRET=loggregator-secret ^
          SYSLOG_HOST_IP=syslog-server.example.com ^
          SYSLOG_PORT=514
```

Special characters must be escaped with `^`.

#### Changing BOSH properties

Note that if BOSH properties are changed in the BOSH manifest, the MSI must be
reinstalled. For example, setting a syslog host in the deployment manifest will
not update the MSI parameters.

## Verify that all the services are up and running

1. If everything has worked correctly, you should now see the
   following five services running in the Task Manager (it's easier to
   sort the services using the `Description` column and look for
   descriptions starting with `CF `):

   | Name          | Description      | Status  |
   |---------------|------------------|---------|
   | Consul        | CF Consul        | Running |
   | Containerizer | CF Containerizer | Running |
   | Executor      | CF Executor      | Running |
   | GardenWindows | CF GardenWindows | Running |
   | Metron        | CF Metron        | Running |
   | Rep           | CF Rep           | Running |
