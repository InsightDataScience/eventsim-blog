# Stress Testing your Spark Code with Terabytes of Streaming Music 

Developing an intuition for how your code scales is one of the trickiest challenges of learning distributed technologies like Spark, Hadoop, or NoSQL databases. Code that works perfectly fine with a modestly sized data set may utterly fail as you reach hundreds of GBs or TBs. This is the cruelty of building a data product: you may not realize that your platform won't scale until you've convinced tens of thousands of users to try it out.  Ironically, this is the worst time to stress test your system, and an excellent recipe for late nights and firefighting.   

To avoid this pitfall, it's critical to test any code that will run on distributed systems with large data sets.  More so, this data set should roughly mimic your use case and distribution of users.  Your testing framework should account for your edge cases and data skew - everything from a surge of Uber requests around 2AM, to the tweet from Justin Bieber that gets 80 million retweets.  

