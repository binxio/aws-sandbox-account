# Protect your personal AWS sandbox account

TLDR; Get a new AWS account, deploy template.yaml using CloudFormation, use the e-mail address and password provided to login, and you're good to go! Oh, and add MFA to your root account and never use it again.

The long version. Somebody created an AWS account for you, or you just registered at amazon.com to start using AWS. Use this blog post to securely setup your personal AWS sandbox account. Even if you don’t have any experience at all, it will guide you step by step through the process. 

It is based on generic AWS security principles, however at a very small scale:

* Your root account is secured with MFA and never used again.
* The setup is written in CloudFormation, "everything as code" principle starts now.
* CloudTrail is enabled for all regions and global services.
* Your personal user account for daily usage only works with MFA, even in the CLI.
* A billing alert is created, so you will receive an e-mail when the estimated bill comes above a threshold you define. This is by default $100/month. 

### Provision your account

First take the following steps. These steps cannot be automated very easily and it's just a one time activity. Probably you even did it already.

1. You're logged in in the console as root, go to the **IAM** Dashboard.
2. Write down the account alias id, we need this in several next steps. Generate an **IAM users sign-in link** and bookmark this link.
3. In the list of checkboxes: **Activate MFA on your root account**.
4. Skip the Create individual IAM users and Use groups to assign permissions for now. We are doing this in the next steps.
5. Apply an **IAM password policy** and remember your policy settings. Because we have password managers, you will probably generate your passwords somewhere using random characters.

Download the [template.yaml](template.yaml) and go to CloudFormation in the AWS Management Console. 

1. Press **Create Stack** and then **Upload a sample template**, select the template.yaml. (Skip until step 10 if you know what you're doing until the stack has been deployed.)
2. Enter a **stack name**, recommended: "<account_alias>-base", *binx-base* in our examples. 
3. **Admin Email** will be your personal email, *jdoe@binx.io* in our examples.
4. Now enter an **Admin Password**, which is a temporary password you will going to change in a minute. But still, it must be 16 chars and meet the Password Policy you just created.
5. **AdminsGroupName**, **AdminsRoleName** have default values. If you have a brand new account I strongly recommend to not change this or to clean up everything before deploying this script.
6. **SpendingAlarm** also has a default of $100, if you think you are going to spend less or more, you could change this value of course. This is just there in case you forget to destroy an (expensive) experiment.
7. Now press **Next**, in this page Click **Advanced** and then enable **Termination Protection**. This prevent you from accidently deleting this base stack in the future. Now press **Next** again.
8. Check the **I acknowledge that AWS CloudFormation might create IAM resources with custom names.** and then press **Create**. 
9. You will receive an e-mail to confirm you are the owner of the e-mail address, this if for the billing notification.
10. Now login to the console using your new password. Now update your password, and to add an MFA device to your user account (go to **IAM**, **Users**, click on your user name, then the tab **Security credentials**, **Assigned MFA device**.) and follow the wizard. Use Google Authenticator or Authy or some other tool to store the virtual MFA.
11. Log out and back in again.
12. Now generate personal **API Keys** and store them somewhere safe.

### Use the console

Login to the console as usual. Click on your user name in the top right of the screen and click **Switch Roles**. Enter your account id and the name of the admins role you entered before. If you didn't change it, the default is: **Admins**. (It's case sensitive.)

### Use the CLI

First we add our personal user account. I often use my e-mail adress, or add a suffix for the AWS account alias, like: jdoe@binx.io@binx. Most consultants have many more profiles to manage, and this is a syntax I prefer.

```
$ aws configure --profile <email>
AWS Access Key ID [None]: <api key>
AWS Secret Access Key [None]: <secret>
Default region name [None]: eu-west-1
Default output format [None]:
```

Then add the following snippet to ~/.aws/config to enable ourselves to assume the AdminsRole we just created in the CLI.

```
[profile <admins_role>@<account_alias>]
role_arn = arn:aws:iam::<account_id>:role/<admins_role>
source_profile = <email>
mfa_serial = arn:aws:iam::<account_id>:mfa/<email>
external_id = <account_id>
region = eu-west-1
```

For example. Notice the uppercase A in Admins, this is indeed case sensitive!

```
[profile admins@binx]
role_arn = arn:aws:iam::123456789123:role/Admins
source_profile = jdoe@binx.io@binx
mfa_serial = arn:aws:iam::123456789123:mfa/jdoe@binx.io
external_id = 123456789123
region = eu-west-1
```

Now verify that your personal account has no access, for example to list S3 buckets.

```
$ aws s3 ls --profile jdoe@binx.io

An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied
```

Now using the admins@binx role, to verify it requests your MFA code, and then does list the S3 buckets. If you run this code again, the MFA token is not requested. By default the aws cli caches your temporary credentials.

```
$ aws s3 ls --profile admins@binx
Enter MFA code for arn:aws:iam::123456789123:mfa/jdoe@binx.io: 
2018-08-12 11:30:39 examplebucket
...
```

Or, if you don't want to type the `--profile` parameter in every command.

```
$ export AWS_DEFAULT_PROFILE=admins@binx
$ aws s3 ls
Enter MFA code for arn:aws:iam::123456789123:mfa/jdoe@binx.io: 
2018-08-12 11:30:39 examplebucket
...
```

Or, if you want to use the profile in a bash or python script (yes, this is AWS cli, but it applies to other scripts as well).

```
$ AWS_DEFAULT_PROFILE=admins@binx aws s3 ls
Enter MFA code for arn:aws:iam::123456789123:mfa/jdoe@binx.io: 
2018-08-12 11:30:39 examplebucket
...
```
