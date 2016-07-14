# Generating Terabytes of Streaming Music Data with Eventsim 

One of the difficult parts of learning distributed systems like Spark, Hadoop, and NoSQL databases is finding a data set to work with that's both large and intriguing.  It should be big enough to test the limits of your system (e.g. it shouldn't all fit in your laptop's main memory), yet rich enough for challenging queries that go beyond cleaning, ETL, and word counts. There are a [number of data sets available](https://aws.amazon.com/public-data-sets/), but finding one that matches the type of application you want to build can be frustrating. 

If you want to test out a low-latency application, there are even fewer options for streaming data, and you don't want to spend time worrying about API rate limits or web scrapers. One solution is to use a script to simulate streaming from a static source of sample data, but then you'll have to spend some time figuring out how to replace the timestamps with relevant times, and also building in logic to account for cycles if you run out of static data.

If you're already going to write a streaming data generator, it may be easier to generate the data from scratch in the first place.  However, you would need to write some logic to give you realistic data based off your use case, which could be non-trivial for even a simple scenario.  For example, generating data for something like a streaming music site (e.g. Spotify or Pandora) would require tracking the state of users, pages, login sessions, artists, songs, and ads, not to mention the relationships between artists and songs.

Fortunately, [Interana](https://www.interana.com/introducing-eventism-the-demo-event-data-generator/) released [Eventsim](https://github.com/Interana/eventsim), a nifty tool for simulating exactly these types of user events. To get started with Eventsim, I'll walk through how you can quickly and easily simulate millions of users for a streaming music site like Spotify.

## Quick Cluster Set Up

To work through a quick example of Eventsim, I'm using our cloud deployment tool, [Pegasus](https://github.com/InsightDataScience/pegasus), to spin up a 4-node Spark/Hadoop cluster on AWS in less than 5 minutes.  Pegasus simply uses the AWS CLI and Bash scripts, and you can easily [set it up with a few steps](http://insightdataengineering.com/blog/pegasus/).  Specifically, I used the following instance configuration for my master

**examples/eventsim/master.yml**

	purchase_type: on_demand
	subnet_id: subnet-d43cb8b0
	num_instances: 1
	key_name: david-drummond
	security_group_ids: sg-faf51d9c
	instance_type: m4.large
	tag_name: davids-eventsim-cluster
	vol_size: 50
	role: master
	
which spins up an on-demand m4.large with 50 GBs of standard EBS storage, and similarly used

**examples/eventsim/workers.yml**

	purchase_type: spot
	subnet_id: subnet-d43cb8b0
	price: 0.13
	num_instances: 3
	key_name: david-drummond
	security_group_ids: sg-faf51d9c
	instance_type: m4.large
	tag_name: davids-eventsim-cluster
	vol_size: 2000
	vol_type: gp2
	role: worker

to spin up 3 spot instances with 2 TBs of general purpose storage (gp2) for each worker.  Installing and starting Hadoop and Spark is as simple as running the following script from the top-level pegasus directory:

**examples/eventsim/spark_hadoop.sh**

	PEG_ROOT=$(dirname ${BASH_SOURCE})/../..
	
	CLUSTER_NAME=davids-eventsim-cluster
	
	peg up ${PEG_ROOT}/examples/eventsim/master.yml &
	peg up ${PEG_ROOT}/examples/eventsim/workers.yml &
	
	wait
	
	peg fetch ${CLUSTER_NAME}
	
	peg install ${CLUSTER_NAME} ssh
	peg install ${CLUSTER_NAME} aws
	peg install ${CLUSTER_NAME} hadoop
	peg install ${CLUSTER_NAME} spark
	
	wait
	
	peg service ${CLUSTER_NAME} hadoop start
	peg service ${CLUSTER_NAME} spark start

## Installing Eventsim

Eventsim is incredibly easy to install, either locally or remotely.  To avoid any dependency issues, I ssh'ed into one of my datanodes (i.e. workers) with

	peg ssh davids-eventsim-cluster 2
	
then cloned the eventsim repo, and ran SBT from within the repo

	git clone https://github.com/Interana/eventsim.git
	cd eventsim
	sbt assembly
	
Note that Eventsim requires Java 8 and Scala, but Pegasus uses an AMI with these dependendencies pre-packaged by default.  Once SBT has done its job, give the eventsim binary execution permission

	chmod +x bin/eventsim

and you're ready to go!

## Getting a Sense of the Data

Let's start by generating a small sample of data for a single user, starting from 7 days ago:

	bin/eventsim --nusers 1 --from 7 -c configs/Accordion-config.json  data/one-user.json
	
Here's a sample of the output in `data/one_user.json`:

	{"ts":1466705356324,"userId":"2","sessionId":5,"page":"NextSong","auth":"Logged In","method":"PUT","status":200,"level":"free","itemInSession":5,"location":"Riverside-San Bernardino-Ontario, CA","userAgent":"\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/36.0.1985.125 Safari/537.36\"","lastName":"Keith","firstName":"Fanny","registration":1466495311324,"gender":"F","artist":"Todd Barry","song":"Sugar Ray (LP Version)","length":126.82404}
	{"ts":1466705454324,"userId":"2","sessionId":5,"page":"Roll Advert","auth":"Logged In","method":"GET","status":200,"level":"free","itemInSession":6,"location":"Riverside-San Bernardino-Ontario, CA","userAgent":"\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/36.0.1985.125 Safari/537.36\"","lastName":"Keith","firstName":"Fanny","registration":1466495311324,"gender":"F"}
	{"ts":1466705482324,"userId":"2","sessionId":5,"page":"NextSong","auth":"Logged In","method":"PUT","status":200,"level":"free","itemInSession":7,"location":"Riverside-San Bernardino-Ontario, CA","userAgent":"\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/36.0.1985.125 Safari/537.36\"","lastName":"Keith","firstName":"Fanny","registration":1466495311324,"gender":"F","artist":"Justin Bieber","song":"Somebody To Love","length":220.89098}
	{"ts":1466705702324,"userId":"2","sessionId":5,"page":"NextSong","auth":"Logged In","method":"PUT","status":200,"level":"free","itemInSession":8,"location":"Riverside-San Bernardino-Ontario, CA","userAgent":"\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/36.0.1985.125 Safari/537.36\"","lastName":"Keith","firstName":"Fanny","registration":1466495311324,"gender":"F","artist":"Sheena Easton","song":"Strut (1993 Digital Remaster)","length":239.62077}
	
This is the activity for a female user from Riverside, CA named Fanny Keith, and we can follow the data as she clicks from songs by Justin Bieber to Sheena Easton. There are sessions that accurately correspond to login and logout events, and it also reflects that she is using the free version with more advertising, but may eventually upgrade to the paid version. Eventsim also includes information on the machine, browser, and the HTTP requests.

The timestamps are measured in milliseconds since the Unix Epoch, and accurately reflect the time at which the Eventsim was ran. The duration between events is pseudo-random, except for the duration for `NextSong` events, which is separated by the length of the previous song. 

All of the song and artist information comes from the [Million Song Dataset](http://labrosa.ee.columbia.edu/millionsong/), while the names and locations are sampled from U.S. Census and Social Security data sets.

### Custom Configurations

The transitions from one state to the next are controlled by the configuration file, which contains entries for each pair of source and destination page, and corresponding probabilities.  For example `Accordion-config.json` contains the following entries for the pages that a logged-in, paid user could visit after seeing an ad:

	{"source":{"page":"Roll Advert","method":"GET","status":200,"auth":"Logged In","level":"paid"},"dest":{"page":"Downgrade","method":"GET","status":200,"auth":"Logged In","level":"paid"},"p":0.05},
    {"source":{"page":"Roll Advert","method":"GET","status":200,"auth":"Logged In","level":"paid"},"dest":{"page":"Roll Advert","method":"GET","status":200,"auth":"Logged In","level":"paid"},"p":0.02},
    {"source":{"page":"Roll Advert","method":"GET","status":200,"auth":"Logged In","level":"paid"},"dest":{"page":"NextSong","method":"PUT","status":200,"auth":"Logged In","level":"paid"},"p":0.8},
    {"source":{"page":"Roll Advert","method":"GET","status":200,"auth":"Logged In","level":"paid"},"dest":{"page":"Cancel","method":"PUT","status":307,"auth":"Logged In","level":"paid"},"p":0.005},
    {"source":{"page":"Roll Advert","method":"GET","status":200,"auth":"Logged In","level":"paid"},"dest":{"page":"Logout","method":"PUT","status":307,"auth":"Logged In","level":"paid"},"p":0.1},
    {"source":{"page":"Roll Advert","method":"GET","status":200,"auth":"Logged In","level":"paid"},"dest":{"page":"Error","method":"GET","status":404,"auth":"Logged In","level":"paid"},"p":0.001},
    
meaning that these users have a

* 80% chance of playing a song
* 10% chance of logging out
* 5% chance of downgrading to a free membership
* 2% chance of seeing yet another ad
* 0.5% chance of completely cancelling their membership
* 0.1% chance of having an error on the site
* 2.4% chance that the session ends

It also has daily and weekly damping effects that simulate reduced activity on nights and weekends with the following options:

	"damping" : 0.09375,
	"weekend-damping-offset" : 180,
	"weekend-damping-scale" : 360,
	"weekend-damping" : 0.50

which means that user activtiy dips sinusoidally, with the minimum 9.375% lower than normal, 180 minutes after midnight,  and the dip lasts roughly  

These configurations enable you to customize the business logic of your expected user activity to a granular level. Eventsim also comes with several convenient pre-built configurations like:

* `Cello-config.json`: Increased weekend-dampening, meaning
that users play somewhat less music than normal on the weekends.
* `Nagara-config.json`: Much higher
advertising rates leading to more downgrades and cancellations.
* `Whistle-config.json`: Happy users with a higher probability of
ThumbsUp events, leading to a higher probability of Upgrade events.

You can learn more about how Eventsim runs the simulations by looking at the main [README](https://github.com/Interana/eventsim/blob/master/README.md), and you can view every pre-built configuration in the [config/README](https://github.com/Interana/eventsim/blob/master/configs/README.md).

### More Users

## Ramping up the Data
hdfs dfs -put - hdfs://nn.example.com/hadoop/hadoopfile

### Configuring the JVM Settings

discuss -XX options, GC, and heap size