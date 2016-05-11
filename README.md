## Notebook
A place for me to take notes on things I encounter throughout my day.
### 2/29/16 - Debuging BI queries
This is a multi-part lesson in hunting down issues.  The issue in question was that some BI requests were (seemingly randomly) stalling for long periods of time.  By long I mean anywhere from 20-90 hours.  Looking at the logs it wasn't obvious why the stalls were happening.  A request would start, queries would run, and then suddenly one of the queries just wouldn't return.  At the same time you would start seeing requests completing hours later than expected.

In order to make any sense of this I had to make sure that all the logs for queries and query responses had the request id included.  Without that it was a meaningless collection of individual queries going out.  Once this was in place you could easily pull up a list of all requests to see what had started and completed.  Once you IDed a request that had behaved oddly you could then grep the logs by that request ID and really track down everything that happened.  A lession in the importance of thorough logging.

So now I had a clear trail to follow in the node logs.  I ran a benchmark test with a concurrency of 4.  I could track that 4 requests were recieved by the node server and track their progress.  All the requests ran fine expect for one that timed out after an hour.  Looking at the logs it was just as before, nothing seemed wrong from node's perspective.  So now I had to look and see Redshift's side of the story.

Redshift has some nice query logging built in so it's pretty easy to track down what queries ran when and what happened with them.  I knew when the 4 queries had gone out (pretty much all at the same time) so I queried the `stl_wlm_query` table to see what queries were queued around that time:
`select * from stl_wlm_query where queue_start_time >= '2016-02-29 18:36:51' and queue_start_time < '2016-02-29 18:36:56' order by query;`

This query showed 4 queries all originating at the same time as expected.  One of them seemed to have ran a little longer than the other three but still finished in under a minute.  Still that's a bit odd since the other queries ran in almost exactly the same amount of time and all 4 queries were suppose to be the same.  With just that information it's odd that node got hung up on one of the queries when Redshift seemed to have completed them all quickly.  Time to look at what these 4 queries actually were on Redshift:
`select * from stl_query where query >= 81821 and query < 81826 order by query;`

The results here were a bit surprising, though they explained the difference in query execution time I saw in the first query.  One of our four queries was a completely different query!  It wasn't even one that our requests would have triggered, it was from a completely different request.  There shouldn't have been anyone else making BI requests (or any requests) on the servers, so it was a bit puzzling to see it.  The biggest red flag to me was that it looked like a 'last_week' interval on an I/O i'd been doing all my BI requets on, but the date range was 2 weeks ago.  It was time to go back to the node logs and see if I could make any sense of this.

Back in the node logs I take a look at the query responses that came in.  I searched for the hash of the unexpected query and found a response hidding with the three expected responses.  Taking a closer look at this query it seems it kicked off at the same time that the other 4 should have.  If anything it took the place of one of the four.  After this query completed it's request continued, finally ended with a total run time of over 87 hours!  Somehow this query just sat around for almost 4 days before making it to Redshift.  The request had long ago timed out, but node's http server hung on to it.

Looking at this more closely it seemed clear that the point at which this query was stalling was while trying to obtain a connection from the PG connection pool.  The query I had expected to run must now be living in that same limbo, waiting for another action to release it and send it on its way.

So now was the issue due to our implementation?  Or a configuration issue?  Or was there are bug in the library?

Starting with the configuration there didn't seem to be any issues or hints of anything that would cause an issue.  Next our own usage of pg's connection pool did have a mistake, though it wouldn't seem to be a fatal one.  We were passing the client back in the `done` method that signaled we were done with a specific connection.  That `done` method expects an error object as the first and only parameter.  If it exists then it assumes something went wrong and it destroys the connection instead of just releasing it.  So our implementation was always destroying connections instead of reusing them.  Definitely not ideal, but also didn't seem like it would cause this issues specifically.

Next I started reading into the `pg` library.  I searched for issues related to connections and connection pooling.  There were plenty, but nothing that looked much like what I was seeing.  I decided to dive into the library to understand how it was working and get an idea of what might be going wrong.  I discovered the `pg` library made use of another library, `node-pool` for it's connection pool logic.  It was in here that all the connection generation, releasing, destroying, and work queuing happened.

The logic was a bit odd.  Work was queued and then calls to a method that would attempt to take on a queued job would immediately get triggered.  Other actions would affect the connection pool or the work queue and almost always also triggered this work method.  It was a bit hard to follow exactly how our issue might happen, but it was convoluted enough that it seemed like it was a suspect.  A quick search through the issues in that library found this open issue: https://github.com/coopernurse/node-pool/issues/30.  Problem solved!  In the end a combination of a library issue and our own mis-implementation.  

I went ahead and fixed our implementation.  So far all tests have been passing and performance has been consistent.  Time to move on to other things!

#### Reference links:
STL_WLM_QUERY Table Reference - http://docs.aws.amazon.com/redshift/latest/dg/r_STL_WLM_QUERY.html
STL_QUERY Table Reference - http://docs.aws.amazon.com/redshift/latest/dg/r_STL_QUERY.html

### 2/18/16 - Learning about Queues in Redshift
Redshift has built in queues for managing queries.  By default there is a single queue with concurrency of 1 for superusers and a second queue with concurrency of 5 for all other users.  You can define your own queues and designate user groups or query groups that determine which queries end up in which queues.  All queries fallback to the default queue if they don't match another.  The max concurrency that user defined queues can have is 50 (combined between all queues).

This is a nice feature to have, but it also means that long queries can back up in the queues if you aren't careful.  You can check on the current query/queue status using some of the following queries.

To see active query status from the default queue (including Running and Queued):
`select * from stv_wlm_query_queue_state where service_class=6;`

#### Reference links:
System Table Reference - http://docs.aws.amazon.com/redshift/latest/dg/t_querying_redshift_system_tables.html


### 12/8/15 - EB struggles, Why Custom AMIs are bad.
Been struggling with ElasticBeanstalk deployments the last couple days.  The bottom line is that custom AMIs should be avoided if at all possible.  They're restricting and don't play well when the platform version changes.  Moreover they can cause really odd issues if they aren't set up properly.

Last Friday I tried to set up a new alpha environment based on our existing production environment (Ops).  When the cloned environment started adding instances those same instances started showing up under the health checks of both the new alpha environment and the ops environment that was the source of the cloning.  Fortunately they didn't start receiving any ops traffic, but they did cause confusion and never would work correctly for the alpha instance.  I tried saving a configuration based on the ops environment and then create a new environment using the configuration.  Same problem.  I tried creating a new environment from scratch, that worked until I changed the AMI to use the AMI from the ops environment.  Then suddenly the same issue.

It was never completely determined exactly why this was happening now, but the root cause was that we were creating the AMI from an instance that was managed by ElasticBeanstalk.  We've definitely done that in the past without encountering these issues.  It's possible that this issue wasn't introduced until the new platform version (we'd recently updated to 2.x).  Either was the solution was to create a custom AMI from a standalone instance.  First I had to figure out what AMI the EB node platform used by default.  Then once I had this I made the standalone instance, did what customizations I needed to on it, and then saved a new AMI.

It solved the problem, but these custom AMIs tend to only work on a specific platform.  So if AWS later updates to 2.0.5 there's a good chance that I won't be able to use the same AMI on environments using the new version.  I'll be forced to remake a new Custom AMI and jump through the same hoops.  This also means that when updated platform version i'll have to create a new environment completely instead of just updating existing ones.  That's a real pain in the ass (I know because I just spent today doing that).

A much better solution is to use the .ebextensions files to download files and run commands to mimick the steps taken to setup the custom AMIs.  In our case we needed GraphicsMagick installed.  Doing some testing it doesn't seem like it's all that complex to have this same setup occur during the deployment process.  Still need to do more tests to make sure it really works well.  If we are able to have it work this way then we won't need to mess around with custom AMIs any more.  This will hopefully make it so we avoid any more odd issues in the future.  It also makes it so we're not tied to any specific EC2 configuration and should be able to change platforms more easily.

In other news I learned that with BASH you can act on inline conditionals using && and ||.  && work as 'then' and || work as 'else'.  So [ -e /usr/local/bin/gm ] && echo "exists" || echo "does not exist";  Will print 'exists' if the file '/usr/local/bin/gm' exists, and 'does not exist' if not.
