+++
author = "Andreas Mosti"
date = 2015-04-25T10:52:10Z
description = ""
draft = false
slug = "updating-a-solrcloud-schema-in-a-live-enviroment"
title = "Updating a SolrCloud schema in a live enviroment"

+++


I can guarantee this will happen to you at some point: You find out that you need to add some more feilds in a Solr schema becouse you want to index some more data from you're documents. Does that mean taking down the live nodes, changing the schema and then start them up again? Thankfully, no. With the help of [Zookeeper](https://zookeeper.apache.org/) and the [Solr schema REST API](https://cwiki.apache.org/confluence/display/solr/Schema+API) we can do this live without any pain. 

###The API calls:

First of, let's update the new config to the zookeeper: 

		./zkcli.sh -zkhost myZookeeper:2181 -cmd upconfig -confname myConfig -confdir /path/to/my/conf/
		
Then just reload the core (I do this on the shard leader out of habit): 

		curl http://mySolrNode:8983/solr/admin/cores?action=reload&core=myCore
		
This trick has saved me many times. 
