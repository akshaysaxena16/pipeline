Q1. What artifacts are generated as part of the Step Functions deployment, and which of these need to be uploaded to S3?
This refers specifically to the artifacts discussed during the call.

Q2. Are the same IAM roles being used by Step Functions across cert and prod environments?
Based on my review, both environments currently use the same IAM roles. Are we planning to continue with this approach, or should we separate roles per environment?

Q3. Which services are primarily invoked by the Step Functions?
From my analysis, the production Step Functions mostly trigger ECS task runs. Please confirm if this is the expected and standard pattern.

Q4. Does the cert environment have a higher number of Step Functions compared to prod?
It appears some Step Functions may exist only for deletion or cleanup purposes. This could lead to pipeline mismatches and make long-term management more difficult—can we clarify the intended design?
