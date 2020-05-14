# Recommendations to AWS

* Throttle calls to create or update trust relationships to limit the ease of enumeration attacks.
* Mask externalID in CloudTrail logs. UserIdentity is masked on failed attempts with HIDDEN_DUE_TO_SECURITY_REASONS. The opposite, logging incorrect externalIDs in clear text but masking the correct value would make sense. Only IAM admins need to know the externalID.
* Add confused deputy warnings to x-acct S3 and CloudFormation documentation.
* Add more detailed guidance specific to 3rd party service providers.
* Add a 12 digit filter to AWS Data Loss Prevention.
* Add GuardDuty checks for abnormal sts:assume role with external id.
* Add CloudWatch alert for externalID which is not a UUID.


