## AWS CloudFormation template to deploy IBM mainframe z/OS on AWS with IBM ZD&T  

This CloudFormation template is provided as part of the AWS Blog Post [Deploying IBM mainframe z/OS on AWS with IBM ZD&T](#).

### Provisioning the Infrastructure  

This CloudFormation template or script allows you to provision all the necessary components to have IBM ZD&T up and running on AWS, with the architecture seen on Figure 1.

![Diagram](imgs/diagram.png?raw=true "Diagram")  
*Figure 1 - IBM ZD&T on AWS architecture overview*

In order to use the provided CloudFormation script you'll need to upload the IBM ZD&T installer and ADCD volumes into an S3 bucket using the following directory structure:

```
+-- s3-bucket-name/
|   +-- installer/
|   |   +-- ZDT_Install_EE_V12.0.4.0.tgz
|   +-- adcd/
|   |   +-- may2019/
|   |   |   +-- may2019_adcd_md5.txt
|   |   |   +-- ADCDTOOLS.XML
|   |   |   +-- volumes/
|   |   |   |   +-- D3BLZ1.gz
|   |   |   |   +-- ... (several other volumes)
|   |   |   |   +-- D3RES1.ZPD
|   |   |   |   +-- ... (several other volumes)
|   |   |   |   +-- SARES1.ZPD
|   |   |   |   +-- ZDTRKT.gz
```

A quick way to upload these files in this structure, is to organize the directory structure on your local machine, then upload it using the following command:  

```
aws s3 cp /local-organized-dir s3://s3-bucket-name/ --recursive
```

After all the files have been uploaded and validated, you can create a stack using the provided CloudFormation template, being careful with the descriptions of the parameters. This template will provision all 3 instances, create a VPC, a Subnet, and all the required components of the solution, including an IAM Role for the Enterprise Server EC2 instance to access and copy the files from the S3 bucket.  
After the stack creation is complete, some processes will still be running in the instances, such as packages updates, installation of the IBM ZD&T binaries, copy of the files from the S3 bucket and others. You can follow along the installation of dependencies, IBM ZD&T packages and volumes being copied by issuing the following command:

```
tail -f /tmp/part-001.log
```

When the initial bash script ends, you'll see the following messages in the logs of each server:

```
# License Server
echo "License Server installation finished, a script scheduled on cron will be checking if the license update files is uploaded to the S3 bucket."
```

```
# Enterprise Server
echo "Enterprise Server installation finished, a script scheduled on cron will be checking if the license_applied file exists in the S3 bucket."
```

```
# Emulator Server
echo "Emulator Server installation finished."
```

Once they all finish, the License Server will have uploaded a file to the S3 bucket with a file name in the following format:  

*hostname*_*unixepochtimestamp*.zip (e.g. ip-10-0-0-38_us-west-2_compute_internal_1602073533.zip)

You then have to send this file to your IBM representative, so that they can generate an update file with the following format:  

*hostname*_*unixepochtimestamp*_update.zip (e.g. ip-10-0-0-38_us-west-2_compute_internal_1602073533_update.zip)

If need be, you can find the complete manual installation instructions in the [IBM ZD&T documentation](https://www.ibm.com/support/knowledgecenter/SSTQBD_12.0.5/com.ibm.zsys.rdt.tools.user.guide.doc/topics/zdt_ee.html).

### Deploying target emulators  

Once you receive the update file from your IBM representative, upload it to the S3 bucket with the same name format, the License Server will be polling for it in 5 minutes intervals. You can check the polling progress and last poll timestamp by issuing the following command on the License Server:  

```
tail -f /tmp/check_license.log
```

The License Server will find the file and apply the license, then it will upload an empty file to the S3 bucket with the name `license_applied`. The Enterprise Server will be polling for this file in the S3 bucket every 5 minutes as well, you can also check the polling progress and last poll timestamp using the same command above, but in the Enterprise Server.  
Once the Enterprise Server finds `license_applied` in the S3 bucket, it will then fire a deploy of the ADCD image it created into the Emulator Server, this may take several minutes. You can check its progress by opening the Enterprise Server Admin Console, by accessing the Outputs section of the CloudFormation stack, and clicking on the URL beside `EnterpriseServerAdminConsoleURL`, with the user and password beside `EnterpriseServerAdminConsoleUser` and `EnterpriseServerAdminConsolePassword` respectively. In the Admin Console, click on `MONITOR AND MANAGE`, then in `TARGET ENVIRONMENTS` you'll be able to expand and check the deployment progress.

> Tip: If you are connected to a VPN or on an office network, it may block port 9443. Try accessing it from outside the VPN or a different network.

![Console](imgs/console.png?raw=true "Console")   
*Figure 2 - IBM ZD&T Admin Console*  

Figure 2 shows the IBM ZD&T Admin Console welcome screen once authenticated.

![Progress](imgs/progress.png?raw=true "Progress")   
*Figure 3 - Verifying the ADCD deployment progress*  

You will be able to see in the screen in Figure 3 when the image has been successfully deployed on the target emulator.
If you choose to deploy a new target emulator manually using the Enterprise Edition web interface, you can find the instructions in the [IBM ZD&T Enterprise Edition User’s Guide](https://www.ibm.com/support/knowledgecenter/SSTQBD_12.0.5/com.ibm.zsys.rdt.tools.user.guide.doc/topics/provisioning.html).  

### Accessing and using z/OS

In order to access z/OS itself, you should use the Emulator Server public IP address, available in the Outputs section of the CloudFormation stack, as well as in the EC2 console itself. The TN3270 service is running on z/OS and listening on ports 23 and 2023, with the latter using a TLS connection and being the one you should use.

![Configuring](imgs/configuring.png?raw=true "Configuring")   
*Figure 4 - Configuring TN3270 terminal emulation connection*  

In order to access z/OS, we use a TN3270 terminal emulator and configure the connection as shown on Figure 4.
 
![Accessing](imgs/accessing.png?raw=true "Accessing")  
*Figure 5 - Accessing the ADCD z/OS*  

Once the TN3270 terminal emulator is configured, we start the connection and see the z/OS welcome screen as shown on Figure 5. 
Since the ADCD z/OS also runs a SSHD service, you can also access z/OS using the same IP as above and port 2022, since port 22 is being used to access the underlying Linux OS running the emulator.
Credentials can be found in the ADCD documentation received together with the volumes.

### Go build

With the IBM ZD&T emulator running z/OS with CICS, DB2, IMS, Websphere Application Server and other products, you can setup your own CI/CD pipeline, start training employees and experimenting!  

If you are interested in deploying IBM ZD&T on AWS and don’t have a license, installer and ADCD volumes, you can contact your IBM representative who will assist you.   

If you would like to have more information related to the use of IBM ZD&T, check [this IBM Redbook](http://www.redbooks.ibm.com/redbooks/pdfs/sg248205.pdf).  

If you need more details on the IBM ZD&T installation steps, feel free to read through [IBM's documentation](https://www.ibm.com/support/knowledgecenter/SSTQBD_12.0.4/com.ibm.zsys.rdt.tools.user.guide.doc/topics/zdt_ee.html).  

Or, if you would like to understand how the ADCD image that is delivered together with the IBM ZD&T license is organized, check [IBM's documentation](https://www.ibm.com/support/knowledgecenter/SSTQBD_12.0.4/com.ibm.zsys.rdt.guide.adcd.doc/topics/t_adcd22_for_zdt.html).   

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

