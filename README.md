<h1>AWS SAP Playbook</h1>

This repository contains code used to provision an entire AWS environment from scratch.  It begins by creating the necessary AWS infrastructure needed to house EC2 Instances.  For example, the first thing the AWS account needs is a Virtual Privat Cloud, or VPC.  Next it needs a Subnet, an Internet Gateway, Route Tables, a Subnet and lastly a Security Group.  After all this is created one can then use the create_vm.yml playbook to provision the EC2 Instance.

Prior to running any of these playbooks someone must first perform the following steps:

1. **Establish an AWS Account** where the infrastructure is to be installed.

1. **Login to the AWS Account** and **Create a new Access Key**

    - To Create a new Access Key, in the upper right corner of the AWS Console you should see a pull down menu connected to the Username that you used to login.  Click on this menu and then select **Security credentials**.

    - On the Security Credentails screen scroll down to the section titled **Access Keys** and click on the **Create Access Key** button.

    - On the Create Access Key screen, select the **Use Case: Command Line Interface (CLI)**, then click the **Next** button.

    - On the Next screen click the **Create Access Key** button.

    - The AWS Console should display a new **Access Key** and **Secret Access Key**.  Copy both these values into a temporary buffer.  These will be used in the next section.

1.  Create a new **Key Pair**.
   
    When creating the new EC2 Instances (or VMs) it's necessary to leverage **Network Security Key Pairs** used when accessing the new VM to make changes.

    To create a new Key Pair follow these steps:

    - Navigate to the **EC2** console window
    - In the upper right corner make sure to select the correct **AWS Region** (i.e. us-east-2) where the infrastructure will be created.
    - Using the menu tabs on the left, select **Network & Security -> Key Pairs** 
    - Click **Create key pair** button
    - Use the following Key Pair input values and click **Create Key Pair**:
      - Name: **SAPKey**
      - Key Pair Type: **ED25519**
      - Private key file format: **.pem**
    - When prompted, store the new Key File locally in the folder of this github repo. 

1.  **Create Encrypted Secret file**

    The Access Key and Secret Access key should be placed in an encrypted file so that the information is safe from prying eyes.  To do this we'll use a tool called **ansible-vault**.  The command to create the secret file is below.  Type this command in a command line terminal.  This command will take you to a linux 'vi' editor.

    ```
    ansible-vault create secrets_file.enc
    ```
    Provide a password for the new Vault file.

    In the Vault file you will need to add the following information performing substitutions to match your environment.  To place the editor in 'insert' mode type the letter 'i'.

    ```
    aws_access_key: <your access key>
    aws_secret_access_key: <your secret access key>
    region: <aws region> # for example us-east-2
    output: json
    root_password: <root password for new EC2 Instances>
    ```

    Once all parameters have been added, click the Escape button to take the editor out of insert mode.  Then type '**:wq**' to write the file and quit out of the editor.

1.  **Create password_file**

    Since the secrets_file.enc is encrypted, we will need to provide a password_file to the Ansible Command Line.  To do this use the 'vi' editor to create a password_file one directory level up.  Populate the password_file with that actual password used when creating the secrets file

    ```
    echo <your_password> >> ../password_file
    ```

1.  Create the **external_vars.yml** file

    The Ansible playbooks are designed to use environment variables when performing the AWS Infrastructure steps.  Use your favorite editor to create the following environment variables in a yaml file:

    ```
    primary_vm: VMHana3
    secondary_vm: VMHana4

    VPC: "TestVPC"
    InternetGateway: "VM-internet-gateway"
    Subnet: "{{ VPC }} - Subnet"
    CIDR: "10.0.0.0/24"
    aws_region: us-east-2
    aws_keypair_name: SAPKey
    aws_instance_size: t2.2xlarge
    tenancy: default
    aws_securitygroup_name: private-network-sg
    aws_vpc_subnet_name: priv-net-private-subnet
    aws_ec2_wait: true
    aws_userdata_template: default

    # Tag Defaults
    hostname: tbd
    vm_name: VMHana3
    vm_deployment: default
    vm_environment: default
    vm_owner: ansible
    vm_purpose: demo
    
    # AMI Lookup
    aws_image_filter: "RHEL-8.6.0_HVM-20230411-x86_64-60-Access2-GP2"
    aws_image_architecture: x86_64
    
    # SAPKey File
    sap_key_file: SAPKey.pem
    sap_domain: "example.com"
    ```

1. **Create the AWS Infrastructure**

    In this step we'll leverage Ansible to create the initial AWS Infrastructure components, specifically the VPC and Network components

    The command to do this is below.  Notice how it takes as parameters the secrets_file.enc, the password_file and the create_vpc.yml playbook.

    ```
    ansible-playbook -e @secrets_file.enc --vault-password-file ../password_file create_vpc.yml
    ```

    After running this command you should now be able to leverage the AWS Console to view the VPC and all of its subcomponents, which include a VPC, Internet Gateway, Route Tables, Subnet and lastly a Security Group.

1.  **Create the new EC2 Virtual Machines**

    In this step we'll run the playbook used to provision the Virtual Machine.  This playbook assumes that that Infrastructure created in the previous step already exists.

    ```
    ansible-playbook -e @secrets_file.enc --vault-password-file ../password_file create_vm.yml
    ```

    After running this playbook you should be able to navigate to the **AWS Console -> EC2 -> Instances** panel to see the new running instance.