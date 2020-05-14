# Recommendations for users of 3rdParty services

* Make a list of all 12 digit AWS account ids you own
* Use Access Analyzer in the IAM portal to find active cross-account trusts. However, it will not find dormant trusts.
* Run scout or cloudmapper weboftrust to gather all the IAM policies against representative accounts (all if possible).
* grep for 12 digit strings grep -oRih -E '(^|[^0-9])[0-9]{12}($|[^0-9])' . | sed 's/[^0-9]//g' | sort -u
* Manually review any IDs from scout/cloudmpaper which are not owned by you
* Examine the externalIDs in trust policies for any values that are not long UUIDs
* It's possible that some of the IDs are directly embedded in S3 or other resource based IAM policies. In those cases, review the policy. There is a high probability of trivial confused deputy attacks
* If the 3rdParty used CloudFormation to create the IAM role, check if it can be deleted or if the roles are overly permissive

## Detection

Create a CloudWatch group with CloudTrail as a source.
Then the following cloudwatch filter will find cross-account assume-roles for further investigation.

```
filter eventName="AssumeRole"
| stats count(*) by requestParameters.roleArn, awsRegion, requestParameters.externalId
```


The following guidance applies to GCP, Azure, AWS and any other public cloud.

* Think very carefully about granting any 3rd party auto-remediate permissions 
* Is there is any chance of a confused deputy attack?
* Do you share a service account credential across projects/resource-groups/accounts? If there is RBAC within the 3rd party service provider portal, do your service accounts granting access to 3rdParty respect the trust boundaries required within the portal. Could someone with access to only GCP project A supply the projectID to project B and gain access? 
