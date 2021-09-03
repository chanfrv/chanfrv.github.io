#### [&#8592; Index](index.md)

# The ultimate guide for a Minecraft server on AWS

While setting up my server with AWS, I encountered a few blogs and forums looking for instructions. However, these posts only give a basic configuration: set up an EC2 with an assigned elastic IP, then SSH on the machine and install by hand the server. I was aiming to create a fully automated infrastructure, accessible from my domain name, resilient to crashes and so on. I went on my way, and I will describe step by step how to acquire my result. The following should be sufficient for a big server to run, where the service provider (you?) has an availability commitment in regard of its players. If you are wondering, I did it out of boredom.

This infrastructure may require a bit of knowledge about Amazon Web Services, as we are about to use a multitude of their services, ohterwise customizing might get a little sketchy.

The overall guide is long, but is mostly point and click so do not be afraid. I will also try to explain what I am doing and why, so that any beginner would kinda understand what is happening.

If I do not precise to set a field, it means that you should keep the default value. Note also that the default Port for a Minecraft server is *25565*, if you do not already have a server, set this value when required.

---

##### Disclaimer:

I do not pretend to be an AWS expert, this may contain potential flaws. It works, however I may have forget to consider some security issues, but it is a Minecraft server after all; your saved world is not exactly sensitive data.

---

## 1. Creating an AWS account

I will not get into details, creating an AWS account is simple and well guided. When your account is created, make sure to follow the [AWS best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) or you might get into trouble.

---

##### Note:

I strongly recommend to enable Multi Factor Authentication (I personnaly use the Android app Google Authenticator). If someone get somehow access to your root or admin account, they will be able to lauch as many instances as they want. And mine bitcoin for example. Expect a salty bill.

---

Once on the AWS console, make sure that your geographical region is selected on the top right corner.

## 2. Network setup

On the top menu **Services** select **VPC**.

### VPC setup

First thing first, we will set up a VPC. This virtual cloud is the sandbox which will contain your whole infrastructure. Simple but mandatory.

On the left panel select **Your VPCs**, then **Create VPC**.

- **Name Tag:** My VPC
- **IPv4 CIDR block\*:** 10.0.0.0/16

Select **Create**.

### Security Group

A **Security Group** defines which traffic can come in and out.

On the left panel select **Security Groups**, then **Create security group**.

- **Security group name\*:** SecurityGroupInstance
- **Description\*:** Allow TCP and UDP access on the instance
- **VPC**: Select your VPC

Select **Create**.

Select your new security group, and on the bottom panel select **Inbound Rules**, then **Edit rules**. Add the following rules:

##### Rule 1:

- **Type:** Custom TCP rule
- **Port range:** Your server port
- **Source:** Anywhere

##### Rule 2:

- **Type:** Custom UDP rule
- **Port range:** Your server port
- **Source:** Anywhere

##### Rule 3:

- **Type:** SSH
- **Port range:** 22
- **Source:** My IP if you plan to connect from a unique computer, otherwise Anywhere

### Subnets Setup

AWS divides the world into geographical **Regions**, each divided in **Availability zones (AZ)**. You can consider each AZ as a server farm. Now that the VPC is up, we must select which AZs we want our server to run on. That is the **Subnet**'s purpose.

On the left panel select **Subnets**, then **Create subnet**.

- **Name tag:** Subnet 1
- **VPC\***: Select the VPC you just created.
- **Availability Zone:** Select the first in the list.
- **IPv4 CIDR block\*:** 10.0.1.0/24

Select **Create**.

I recommend creating a second subnet, it is a good practice whatever you are running on your instances. If one of the AZ is down (meaning a server farm is down for some reason), your instance will be removed from the first subnet and restarted on the other. This part is completely optional, but still I recommend it and it will not cost you extra (Amazon encourages good practices like this).

#### Second subnet setup

- **Name tag:** Subnet 2
- **VPC\***: Select the VPC you just created.
- **Availability Zone:** Select the second in the list.
- **IPv4 CIDR block\*:** 10.0.2.0/24

Now right click on both subnet (one after the other don't try the devil) and select *Modify auto-assign IP settings* and check **Auto-assign IPv4**.

### Internet Gateway

The **IGW** is a component that allows communication between instances in your VPC and the internet. Pretty useful.

On the left panel select **Internet Gateways**, then **Create internet gateway**.

- **Name Tag:** MyInternetGateway

Hit **Create**.

Once created, right click on the new IGW, select *Attach to VPC* and select your VPC.

### Route table

The **Route table** defines where the incoming traffic goes.

On the left panel select **Route tables**, then **Create route table**.

- **Name tag:** Route Table
- **VPC\***: Select the VPC you just created.

Select **Create**.

On the bottom panel select **Subnet Associations** and then **Edit subnet associations**. Select both subnets you just created and hit **Save**.

Again, on the bottom panel, on **Routes**, select **Edit routes** and then **Add route**.

- **Target:** Your internet gateway *igw-XXXXXXXX*
- **Destination:** 0.0.0.0/0

Press **Save routes**.

## 3. Elastic File System

An **EFS** is a network filesystem which can be plugged in any instance with its *ID*. It only require to mount the filesystem, it is as easy as using a USB key to carry the saved files around the different instances.

### Security Group

First, create a **Security Group**.

- **Security group name\*:** SecurityGroupEFS
- **Description\*:** Allow EC2 access on the EFS
- **VPC**: Select your VPC

Select **Create**.

Select your new security group, and on the bottom panel select **Inbound Rules**, then **Edit rules**. Add this new rule:

- **Type:** NFS
- **Port:** 2049
- **Source:** SecurityGroupInstance

Now select the security group *SecurityGroupInstance* and add the following rule:

- **Type:** NFS
- **Port:** 2049
- **Source:** SecurityGroupEFS

### EFS Setup

On the **Services** menu, select **EFS** and then **Create file system**.

- **VPC:** Your VPC
- **Create mount target:** Make sure both subnets are selected with the security group *SecurityGroupEFS*.

Select **Next step**, and **Next step** again, and finally **Create file system**.

Once you file system created, keep the **File System ID** written somewhere, it should look like *fs-XXXXXXXX*.

## 4. Server setup

On the top menu **Services** select **EC2**.

### Key Pair

You can generate a set of keys to access the EC2 via SSH, which is useful to test or update the server.

On the left panel select **Key Pairs**, the creation is straightforward.

Once created, the console asks to download the key with a *.pem* extension to you computer. It allows to SSH from your terminal, but is not mandatory: AWS comes with a web console to access your instance via SSH.

Once your instance will be running, on the left panel **Instances**, select your running instance, select **Connect**, choose **EC2 Instance Connect (browser-based SSH connection)**, leave the user name as it is (which should be *ec2-user*) and **Connect**.

### Load balancer

Let's first talk quickly about **Route 53**. This service allows to link your domain name (even if it is not hosted by AWS) to you architecture. We will come back to this at the end. **Route 53** will require a *unique* entry point to our architecture. You would usually link it to a EC2 instance, but ours will be autogenerated and is subject to change, that is why we will use an **Elastic Load Balancer (ELB)** (yes everything is elastic on AWS). Its sole purpose will be to serve as entry point, find the running instance in our **Subnets** and redirect the traffic towards it.

On the left panel select **Load Balancer** and then **Create Load Balancer** (They are so inconsistent on capital letters that's annoying).

Select the **Network Load Balancer (NLB)** as we are using the TCP and UDP protocols.

- **Name:** My NLB

In **listeners**:

- **Load Balancer Protocol:** TCP_UDP
- **Load Balancer Port:** Your server port

In **Availability Zones:**

Select your **VPC**, enable both **Availability zones** and select the associated subnets you created.

Select **Next: Configure Security Settings**, then **Next: Configure Routing**.

Here create a new **Target Group**, you just have to fill the field **Name:** MyTargetGroup

Select **Next: Register Targets**, **Next: Review** and finally **Create**.

### Launch template

Now we get on the part where we tell AWS "I want my instance to get this OS, this much RAM, execute this script when you launch it" and so on.

On the left panel select **Lauch templates** and then **Create launch template**.

- **Launch template name\*:** MyLaunchTemplate
- **Template version description:** EC2 small, jdk 1.8.0
- **AMI ID:** Select *Search for AMI*, **AMI catalog:** Quick Start, **AMI:** Amazon Linux 2 AMI
- **Instance type:** t2.small

---

##### Note:

You will have to consider which instance type to pick depending on the amount of players the server will host. For up to 10 players a *t2.small* is enough, up to 25 players a *t2.medium*... You may have to check by yourself by testing different instance types. I would not recommend *t2.micro* (which is a shame because that's free tier) as they are really not powerful enough, even for few players.

---

- **Key Pair name:** The one created earlier
 - **Security Groups:** Select the group created earlier.
 
Expand **Advanced details**.

In **User data** (the script which will be launched when the server is turned on), paste the following:

```
#!/bin/bash

# updating and downloading java
yum update -y
yum install -y java-1.8.0-openjdk.x86_64

# mounting EFS
yum install -y amazon-efs-utils
mkdir efs
mount -t efs fs-XXXXXXXX:/ efs
cd efs

# update server version
[[ -d minecraft_server ]] || mkdir minecraft_server
cd minecraft_server

wget https://launcher.mojang.com/v1/objects/3dc3d84a581f14691199cf6831b71ed1296a9fdf/server.jar -O server.jar

if [ ! -f "eula.txt" ]; then
  # first run
  java -jar server.jar nogui
  # you must agree to eula bla bla bla
  sed -i s/eula=false/eula=true/g eula.txt
fi

# run
java -Xms1024M -Xmx1024M -jar server.jar nogui

# remove server, we download it each time anyways
rm server.jar
# lazy does not mean disgusting
cd /
umount efs

# shutdown if the server is down, auto scaling will turn it back on
shutdown -h
```

You may notice the line `mount -t efs fs-XXXXXXXX:/ efs` is incomplete. Replace `fs-XXXXXXXX` with the **File System ID** you kept.

If you got a bigger instance type than *t2.small*, remember to change the line `java -Xms1024M -Xmx1024M -jar server.jar nogui` and adjust `-Xmx1024M` depending on the total RAM available (I recommend leaving 1GB free for the system to run correctly).

I download the server it each time, which is not very sexy, but changing the server version will be way easier. Just change the url used by `wget` in a new template version.

---

##### Open letter to Mojang:

> Dear Mr. and Ms. Mojang,
>
> Today I went to see Father Christmas at the mall, and I asked for a persistent link to the last server version, but he told me that was not possible because the Chrismas elves were very busy. I have been very kind this year so I was hoping you could help me.
>
> Much love,
>
> chanfrv

---

Select **Create launch template**.

### Auto Scaling group

**ASG** is one of the coolest features of AWS: instances will be automatically generated/terminated depending on the traffic. Because our server requires everyone to be on the same server (otherwise players would not see each other), we won't use the full capabilities of this feature. However, we will ask AWS to maintain the server count to 1. That way, if the server is down, the auto scaling will turn an other machine on.

On the left panel, select **Auto Scaling Groups**, then **Create Auto Scaling group**.

Select **Launch template** then select the template you just created, then hit **Next step**.

- **Group name:** MyASG
- **Launch Template Version:** Default

---

##### Note:

By setting the version to *Default*, if you need to change the **Launch template**, for example because you want to change the jdk version, you will just have to set the new **Launch template** as default (right click on it) and the next instance created by the auto scaling will have the new version.

---

- **Network:** You VPC
- **Subnet:** Select both

Expand **Advanced details**.

- **Load Balancing:** Check *Receive traffic from one or more load balancers*
- **Target Groups:** Select the one created while creating the **Load Balancer**, there should be one, take it.
- **Health Check Type:** ELB

Select **Next: Configure scaling policies**.

Leave **Keep this group at its initial size**, select **Review** and finally **Create Auto Scaling group**.

Now, you can go on **Instances** on the left panel and you should see shortly a server being created. Finally!

No, come back here, it is not over yet.

First, wait a few minutes for the **Status check** to end. If you have the result *2/2 check passed*, it means that your instance received a ping from the **ASG** and responded. The server is running, that's a start.

To finally test if the server works, on the left panel select **Load Balancers**, select the load balancer you just created, and on the bottom panel in **Description** copy the **DNS name** which should look like *ELB-XXXXXXXX.elb.my-region.amazonaws.com*. Launch the Minecraft client and test a direct connexion *LoadBalancerDNS:Port*, the port being the one chosen in the **Target Group**, which should also be the one selected on the server (you can SSH on the machine and check by yourself).

## 5. Route 53

As I said, **Route 53** allows to link your domain name and the architecture, in our case directly on the **Load Balancer**. I can see 3 different cases you're in at this point, pick one of them.

On the **Services** menu select **Route 53**.

### What ? Route 53 is not free ? And you have to pay for domain names ? Forget it bud.

Then you have your DNS name from your **ELB**, it is not very good looking but it works.

### I do not have a domain name

If you are willing to pay for a domain name (which you will be able to use for other applications or websites), on the dashboard select **Register domain** and follow the instructions. Then in the left menu select **Hosted Zone**, and **Create Hosted Zone**.

- **Domain Name:** Your fresh new domain
- **Type:** Public Hosted Zone

Hit **Create**.

Then skip the next part and go to the **Record Set** part.

### I already have a domain

On the left menu select **Hosted Zone**, and **Create Hosted Zone**.

- **Domain Name:** Your good old domain
- **Type:** Public Hosted Zone

Hit **Create**.

Select your new hosted zone. 2 entries should appear: 4 different addresses, these are the **Name servers**, while the other one is the **SOA – Start of authority**. On you DNS provider application, in the DNS settings, you should have a list of **Name servers**. Select **Use custom name servers** and add the 4 Name servers provided by AWS.

### Record Set

You domain name is linked to **Route 53**, it must now be linked to the architecture. Select you new **Hosted Zone** and select **Create Record Set**.

- **Name:** You can leave it empty, which would make the Minecraft server using the default domain name. If you are hosting a website, considering that a web request uses the ports 80, 8008 and 8080 there should be no conflicts if they use the same domain name.
- **Type:** A
- **Alias:** Yes
- **Alias Target:** Select your ELB under *ELB network load balancers*.

Hit **Create** (Last one I promise).

You should now be able to access your server via the Minecraft client using *YourDNS:Port*.

## 6. Backups (Optional)

A nice addition would be versionning: if someone messes up your server, you would always have previous versions to fall back on.

### Duplicity on the EFS

Duplicity is an encrypted incremental backup to local or remote storage. Add the following in the **User data**, before launching the server:

```
yum install -y duplicity
cd /efs
[[ -d backups ]] || mkdir backups
duplicity --no-encryption --exclude minecraft_server/server.jar minecraft_server backups
cd minecraft_server
```

If someone blew on your wooden house, first, consider bricks, then run `sudo duplicity /efs/backups /efs/minecraft_server` and you're good to go. As your instance may run for a long time, you could schedule recurrent backups by adding this function in the **User Data**:

```
recurrent_backup()
{
  while true; do
    # see sleep(1)
    sleep 1h
    duplicity --no-encryption --exclude minecraft_server/server.jar minecraft_server backups
  done
}
```

And this line before lauching the server: `recurrent_backup &`.

Have a look at **duplicity(1)**, you can encrypt, upload to a distant server (so why not to a separated s3 bucket), select the version, the specific files to version, and more.

## 7. Hosting multiple servers

Each new server hosted on the same domain will require:

- Edit your **Security Group** and add the new port in **Inbound Rules**.
- A new **Target Group** which will accept the new server port.
- In **Load Balancers**, select your **NLB**, in the bottom panel **Listeners**, **Add listener**, and add a *TCP_UDP* listener on the new port redirecting to the new **Target Group**.
- A new **Elastic File System** with the same creation procedure and a new **Launch Template** (using an older one as model), where you would have to change the *File System ID* in **User data**. You might have to SSH into a dummy instance, mount the **EFS**, launch the server for the first time and change the Port in *server.properties*.
- A new **Auto Scaling group** including the new **Target Group** and **Launch Template**.

## 8. Optimize Costs

To optimize costs, a solution would be to use **Spot instances**, which are selected either while creating the **Template Instance** or the **Auto Scaling group**. **Spot instances** are unused instances by AWS, which mean that using them will be way cheaper. However, this instance may be terminated at any moment, so expect a few downtime. On the other hand, auto scaling will handle the downtime without you touching anything.

If, like me, you plan to leave your server constantly running, you should consider **Reserved instances**. You will have to reserve a specific instance type (so the same as the one in the **Launch Template**), pay up-front or partially, and your running instance will be free. An *way* cheaper. You will however have to consider how long your Minecraft hype will last: if you reserve an instance for a year and stop playing after a week, well...

---

Thank you for coming to my Ted Talk.

Special thanks to r/aws for their unvaluable advices.
