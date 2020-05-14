# Guide for Vendors Implimenting AWS Integrations Using Cross-Account Assume-Role with ExternalId

## Use minimal scoping of IAM permissions

* Prefer a specific role prefix rather than “root” in the trust policy of the role in the client account to be assumed. When using “root” any principal in the 3rd party AWS account with sts:assumeRole on * could assume the client’s role. Instead, use “role/<3rdParty-name-service>” in the trust policy.
* For SOARs and other automated remediation products, do not request permissions that will require human review anyway. For example, it may be reasonable to remove non-whitelisted security groups or ingress rules in violation of policy automatically. 
** ec2:DeleteSecurityGroup - OK
** ec2:RevokeSecurityGroupIngress- OK
* But rather than granting permission to modify a policy arbitrarily, consider just generating the suggested policy which requires no permissions.
** ec2:CreateSecurityGroup - Consider carefully. Prefer generating a policy and letting a human apply with their own credentials.
*When using CloudFormation templates to install resources to a client, take great care that resources created do not introduce un-necessary privileges like invoking lambdas and modifying IAM arbitrarily.


## Treat external_id like you would an ssh private key in the UI and backend

* Mask it from the user.
* Do not include it in the body or URL parameters of any web posts.
* Do not return it in the rest response unless explicitly requested by the user.
* Do not log the externalID in clear text.
* Assume XSS, CSRF, and SSRF are present in the web application.
* Ensure that all traces of externalIds and customer AWS account IDs are removed when customers are off-boarded or integrations removed.
* Despite the fact that AWS documentation says that in the context of the AWS customer account they do not treat the externalID as a secret, in the context of the 3rd party portal, it absolutely should be treated like a secret.


## Prevent unauthenticated enumeration attacks and tampering

Remember, your AWS account ID is already public as part of the integration assume-role trust process and many of your customers leak theirs.

* Add random suffixes to any role, policy, user or S3 bucket in your account.
* Scan for 12 digit AccountIDs (or GCP or Azure identifiers) in git repos and public documentation and presentations (slide-decks, screenshots). Try not to expose any accountIDs besides the one used for the integration.
* Add a random suffix to any IAM, Lambda or other CloudFormation resources established in a customer’s account.
* Ensure the API endpoints for account on-boarding are rate-limited.
* Ensure that the API backing the UI for connector creation does not accept ExternalID as a parameter. If so, any UI controls can be bypassed by intercepting calls from the UI to the API and modifying externalID.


## Support automation and trust boundaries for many accounts

* Ideally, support a terraform or ansible provider that allows for complete automation of AWS IAM policies.
* Create a unique externalID per account to support RBAC within the portal. Eg. devs only access dev account. Admins access prod and dev. Using an externalID per customer does not support this.
* Have an exposed API endpoint and UI button that generates/rotates the externalID.
* Companies with 40+ accounts will want to automate AWS account integration on-boarding. Consider an endpoint that accepts a list of account_ids and returns a map of {account_id: external_id} for friendly on-boarding of many accounts. Facilitate rotating externalIDs. Offer an API call like /integrations/aws/{integration_id}/externalid/regenerate. The customer supplies their admin credentials to 3rdParty to call the endpoint and regenerates a hard UUID. Ideally, 3rdParty could create a custom provider in terraform with a “regenerate: true” field for the field. If 3rdParty offers a deployment by CloudFormation template, include a script that accepts the template and a list of AccountIDs to generate and execute all the templates in AWS. (MongoDB DataLake has a good example of using the CLI instead of CFN). 

### Customer Notification
If the IAM policy generated was generally correct and a cryptographic UUID was generated for the customer. However, if the user was allowed to choose their own externalID, it was a static value, or no externalID was set in the recommended policy, then to best protect the customer, those with weak externalIDs should be notified to rotate their externalID to a stronger value. The vendor has all the externalIDs and can audit them for strength and only notify those with weak values. Preventing users who create new accounts should protect against the externalID attack, but other vulnerabilities like SSFR or XSS could allow an attacker unhindered access to customer AWS accounts. However, for IAM assume-role attacks, after fixing the modifiable externalID issue, it is possible not to notify the customer.

Direct Resource Access issues are those where a third party AWS account ID is granted access to a resource’s IAM policy (like a bucket policy). In these cases, there is no externalId available. The correct means of granting access to third parties is to have the vendor assume the role in the customer account and then have the assumed role access the resource. If the vendor accountID is found in any S3 or other resource policy, the policy must be changed and customers must be notified. A CVE is recommended.
If the assume-role is deployed via a CloudFormation template, the template should be reviewed for any resources deployed into the customer account. If excess-privileged resources were left over, customers should be notified with a CVE to clean-up the resource, or a stack modification update should be added to the CFN stack to fix the issue.
