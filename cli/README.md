# Manage virtual machines with the Azure CLI
For more https://aka.ms/AzureCLI3

## Intro

The Azure Portal is great for one-off tasks but when it comes to multiple tasks, navigating through the various panes adds time and isn’t productive. 

This is where the command line shines! You use commands and scripts to run repetitive tasks.

With Azure, you have two different command lines tools you can work with:

- Azure PowerShell (Windows admins tend to prefer this one)
- Azure CLI (Linux admins tend to prefer this one)

With either one, you can write scripts to check the status of cloud servers, deploy new configurations, open ports in the firewall, or connect to a virtual machine to change a setting.

In this session we are going to focus on the Azure CLI, but any task can be accomplished in the PowerShell version. 

## What is the Azure CLI?

The Azure CLI is Microsoft’s cross-platform command-line tool for managing Azure resources. You can use it on Mac OS, Linux, Windows and from a browser via the Azure Cloud Shell.

In this session we’re going to be working with Virtual Machines, let’s start first with creating one.

# Creating a virtual machine with the Azure CLI

With the azure CLI, every command will start with `az`

The `vm` command is used to work this virtual machines in the Azure CLI. Some of the most common sub commands are:

- `create`
- `deallocate`
- `delete`
- `list`
- `open-port`
- `restart`
- `show`
- `start`
- `stop`
- `update`

For a complete list of commands, check out [Azure CLI reference documentation](https://docs.microsoft.com/cli/azure/reference-index?view=azure-cli-latest). 

Let’s start with the first one: `az vm create`

# az vm create

This command is used to create a virtual machine in a resource group. The four parameters that must be supplied are:

- `--resource-group` The resource group that will own the virtual machine.
- `--name` The name of the virtual machine. Must be unique within the resource group.
- `--image` The operating system image to use to create the VM.
- `--location` The region to place the VM in. Typically this would be close to the consumer of the VM.
- `--verbose` Helpful to see progress while the VM is being created.

Additionally we'll provide these parameters:

- `--admin-username` We are specifying the administrator account. By default the command will use your current username but since the rules for account names are different for each OS, it's safer to specify a name.
- `generate-ssh-keys` This parameter is used for Linux distributions and creates a pair of security keys so we can use the ssh tool to access the virtual machine remotely. The pair of files are placed into the `.ssh` folder on your local machine and in the VM.
    
    If you already have an SSH key named `id_rsa` in the target folder, it will be used rather than having a new generated key. 
    

Let's put it all together.

```bash
az vm create --resource-group reactorclidemo-rg --location eastus2 --name reactorclivm1 --image UbuntuLTS --admin-username azureuser --generate-ssh-keys --verbose
```

Once the command is done, you will get a JSON response which includes the current state of the virtual machine and its public and private IP addresses assigned by Azure. 

# Connect to our VM with SSH

We can test that the Linux VM is up and running by using the public IP address in the ssh tool. We generated an SSH key pair when we created the VM, so we don't need a password. Since we set out admin name to `azureadmin` we can use the following command to connect

```bash
ssh azureuser@publicipaddress
```

# VM images

We used the `UbuntuLTS` image but Azure has several standard VM images we can use. To list them, use the `az vm image list --output table` command. This will output the most popular images that are part of an offline list built into the Azure CLI. 

Keep in mind that you can also create and upload your custom images to create VMs based on unique configurations. 

If you want to get a full list, add the `--all` flag to the command. It helps to filter the list with the `--publisher` `--sku` or `--offer` options. 

If we want to see all Wordpress images, we can use

```bash
az vm image list --sku Wordpress --output table --all
```

Some images are only available in certain locations. You can add the `--location` flag to the command to scope the results to ones available in the region where you want to create the virtual machine. 

```bash
az vm image list --location eastus2 --output table 
```

# Sizing VMs properly

A VM without the correct amount of memory or CPU will fail under load or run too slowly to be effective. 

Azure defines a set of pre-defined VM sizes for Linux and Windows to choose from. The available sizes change based on the region you're creating the VM in. You can get a list of available sizes using the `az vm list-sizes` command. 

We didn't specify a size in our create command, Azure selected a default general-purpose size for us. We can specify the size using the `--size` parameter. We could create a 2-core virtual machine:

```bash
az vm create --resource-group reactorclidemo-rg --name reactorclivm1 --image UbuntuLTS --admin-username azureuser --generate-ssh --verbose --size "Standard_DS2_v2"
```

We can also resize an existing VM. We first have to check if the desired size is available in the cluster our VM is part of. 

`vm list-vm-resize-options`

```bash
az vm list-vm-resize-options --resource-group reactorclidemo-rg --name reactorclivm1 --oputput table
```

This will return a list of all the possible size configurations available in the resource group. If the size we want isn't available in our cluster but is available in the region, we can deallocate the VM. This would stop the running VM and remove it from the current cluster without losing any resources. We can then resize it. 

To resize a VM, we use `vm resize` 

```bash
az vm resize --resource-group reactorclidemo-rg --name reactorclivm1 --size Standard_b2s
```

# Query system and runtime information about the VM

We can use other vm commands to get information about our newly created machine. 

- `az vm list` will return all machines defined in this subscription. Filter the output with `--resource-group`
- We've been using the `--output` flag with `table` this is to format the output in a way that's easier to read than the default JSON.
- `az vm list-ip-addresses` will list the public and private IP addresses for a VM.
- `az vm show` will give us more detailed information about a specific VM. This will return  fairly large amount of JSON data including attached storage devices, network interfaces, and more. This is a great chance to use JMESPath, a built-in query language for JSON.

# Filter our Azure CLI queries

```bash
az vm show --resource-group reactordemocli-rg --name reactorclivm1 --query "osProfile.adminUsername"
```

You might also find it help to pair with the `--output tsv` parameter. It is useful for scripting as well - for example, you can pull a value out of your Azure account and store it in an environment or script variable.

# Start and stop your VM with the Azure CLI

## Stop

We can stop a running VM with the vm stop command. You must pass the name and resource group, or the unique ID for the VM:

```bash
az vm stop \
--name SampleVM \
--resource-group [sandbox resource group name]
```

We can verify it has stopped by attempting to ping the public IP address, using ssh, or through the vm get-instance-view command. This final approach returns the same basic data as vm show but includes details about the instance itself. Try entering the following command into Azure Cloud Shell to see the current running state of your VM:

```bash
az vm get-instance-view \
    --name SampleVM \
    --resource-group [sandbox resource group name] \
    --query "instanceView.statuses[?starts_with(code, 'PowerState/')].displayStatus" -o tsv
```

This command should return VM stopped as the result.

## Start

We can do the reverse through the vm start command.

```bash
az vm start \
    --name SampleVM \
    --resource-group [sandbox resource group name]
```

This command will start a stopped VM. We can verify it through the vm get-instance-view query, which should now return VM running.

## **Restart a VM**

We can restart a VM if we have made changes that require a reboot running the `vm restart` command. You can add the `--no-wait` flag if you want the Azure CLI to return immediately without waiting for the VM to reboot.

## **Delete a VM**

```bash
az vm delete \
    --name vm1 \
    --resource-group rg
```

# Installing software on your VM

1. Locate the public IP address of your SampleVM Linux virtual machine.

```bash
az vm list-ip-addresses --name SampleVM --output table
```

2. Next, open an ssh connection to SampleVM.

```bash
ssh azureuser@<PublicIPAddress>
```

3. Once you are logged in to the virtual machine, execute the following command to install the nginx web server.

```bash
sudo apt-get -y update && sudo apt-get -y install nginx
```

4. Exit the Secure Shell.

```bash
exit
```

5. In Azure Cloud Shell, use curl to read the default page from your Linux web server using the following command, replacing <PublicIPAddress> with the public IP you found previously. Alternatively, you can open a new browser tab and try to browse to the public IP address.

```bash
curl -m 10 <PublicIPAddress>
```

6. This command will fail because the Linux virtual machine doesn't expose port 80 (http) through the network security group that secures the network connectivity to the virtual machine. We can change this with the Azure CLI command vm open-port.

7. Type the following into Cloud Shell to open port 80:

```bash
az vm open-port \
    --port 80 \
    --resource-group [sandbox resource group name] \
    --name SampleVM
```

It will take a moment to add the network rule and open the port through the firewall.

8. Run the curl command again.

```bash
curl -m 10 <PublicIPAddress>
```

9. This time it should return data. You can see the page in a browser as well.

```bash
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
body {
    width: 35em;
    margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif;
}
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```