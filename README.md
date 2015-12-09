## Notebook
A place for me to take notes on things I encounter throughout my day.
### 12/8/15 - EB struggles, Why Custom AMIs are bad.
Been struggling with ElasticBeanstalk deployments the last couple days.  The bottom line is that custom AMIs should be avoided if at all possible.  They're restricting and don't play well when the platform version changes.  Moreover they can cause really odd issues if they aren't set up properly.

Last Friday I tried to set up a new alpha environment based on our existing production environment (Ops).  When the cloned environment started adding instances those same instances started showing up under the health checks of both the new alpha environment and the ops environment that was the source of the cloning.  Fortunately they didn't start receiving any ops traffic, but they did cause confusion and never would work correctly for the alpha instance.  I tried saving a configuration based on the ops environment and then create a new environment using the configuration.  Same problem.  I tried creating a new environment from scratch, that worked until I changed the AMI to use the AMI from the ops environment.  Then suddenly the same issue.

It was never completely determined exactly why this was happening now, but the root cause was that we were creating the AMI from an instance that was managed by ElasticBeanstalk.  We've definitely done that in the past without encountering these issues.  It's possible that this issue wasn't introduced until the new platform version (we'd recently updated to 2.x).  Either was the solution was to create a custom AMI from a standalone instance.  First I had to figure out what AMI the EB node platform used by default.  Then once I had this I made the standalone instance, did what customizations I needed to on it, and then saved a new AMI.

It solved the problem, but these custom AMIs tend to only work on a specific platform.  So if AWS later updates to 2.0.5 there's a good chance that I won't be able to use the same AMI on environments using the new version.  I'll be forced to remake a new Custom AMI and jump through the same hoops.  This also means that when updated platform version i'll have to create a new environment completely instead of just updating existing ones.  That's a real pain in the ass (I know because I just spent today doing that).

A much better solution is to use the .ebextensions files to download files and run commands to mimick the steps taken to setup the custom AMIs.  In our case we needed GraphicsMagick installed.  Doing some testing it doesn't seem like it's all that complex to have this same setup occur during the deployment process.  Still need to do more tests to make sure it really works well.  If we are able to have it work this way then we won't need to mess around with custom AMIs any more.  This will hopefully make it so we avoid any more odd issues in the future.  It also makes it so we're not tied to any specific EC2 configuration and should be able to change platforms more easily.

In other news I learned that with BASH you can act on inline conditionals using && and ||.  && work as 'then' and || work as 'else'.  So [ -e /usr/local/bin/gm ] && echo "exists" || echo "does not exist";  Will print 'exists' if the file '/usr/local/bin/gm' exists, and 'does not exist' if not.
