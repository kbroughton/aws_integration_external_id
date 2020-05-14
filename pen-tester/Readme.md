# Recommendations to Pen-Testers

## Basic Testing for Vulnerabilities
* SSRF, CSRF become critical findings if the externalID can be tampered with.
** SSRF in particular requires particular care. If the web-service is run on an EC2 instance which has the role used for assume-role into clients AWS accounts, then access to AWS metadata service would lead to theft of the Deputy’s credentials. This is no longer Confused Deputy, but Robbed Deputy which is much worse as the attacker is no longer constrained by the actions defined by the vendor. 
* Any cloud provider could be a victim of 3rd party confused deputy if accountID, GCP project ID, Azure subscription etc are not checked to belong to a user when performing actions. Can you tamper to confuse the deputy?
* Are there any direct access grants to resource (like S3 buckets)? These don’t have externalID support and are usually immediately vulnerable to a confused deputy attack. Resources should be accessed via an assumed role or with user credentials not by whitelisting an external account in the bucket policy.
* Is externalID treated like a private key? (masked from user, revealed on request via separate fetch, not just returned like the rest of the page data).
* Are there any url paths, parameters, or POST bodies that include externalID and can they be tampered with? The UI may make not allow the user to modify the externalID, but the API backing the UI accepts externalID as a parameter, the API is likely still vulnerable.
* Are the externalIDs cryptographic UUIDs? Is there a mechanism to rotate them
* If you think you have found a vulnerability, create a second account to ensure there are not alternative protections.
** Some vendors have chosen to address the confused deputy by allowing customers to choose externalId, but preventing a user from enrolling an AWS account already added by someone else. 
** There are several concerns with this approach such as dev and test DBs in the same AWS account and offboarding/life-cycle management issues.

## Defense-in-depth or  Whitebox Testing

* do they support for automation patterns?
* do they support RBAC to match the customer’s RBACs between AWS accounts? 
** Large enterprises might want different privilege levels within the vendor portal.
** Lower privilege roles should not be able to onboard new AWS accounts or confused-deputy is likely
* Test for code injection. AccountID, roleArn, externalId are passed to a call that looks like `aws sts assume-role --role-arn $roleArn --external-id externalId --role-session-name theirvalue` or the equivalent in a python/java/js sdk. Do they sanitize the input? If it’s a bash cli, you might be able to put spaces in the ARN and end with #. The aws cli will take the last option as the correct value.
* Though we will not try to brute-force the ExternalId of the 3rd party vendor without their permission, there are two cases which we should test for:
** XXXXXXX release at a later date
** If the user can choose the name of the role, then the 3rd party role which calls sts assume-role likely has the permission sts:AssumeRole on Resource “*”. What is preventing us from assuming common role names (especially managed roles like AdminstratorFullAccess) in the vendor’s own account? Probably nothing. Insert golum emoji here. It would require them to write an explicit deny for their own account, like Effect=Deny, Action=sts:AssumeRole, Resource=arn:aws::
* What is their lifecycle management? A potential customer could sign up for free trial, add the role to their account, then not pay for the service. Does the vendor delete their account from the DB? If so, and they didn’t remove the IAM role manually, they are vulnerable. If not, nobody else from the org can add that account at a later date.

 
