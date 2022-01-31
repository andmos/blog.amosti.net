+++
author = "Andreas Mosti"
date = 2015-04-13T18:51:36Z
description = ""
draft = false
slug = "zookeeper-aware-application-server"
title = "Zookeeper-aware application server"
+++



###The situation
At work we use [Apache Solr](http://lucene.apache.org/solr/) from our application server to index and offer search in documents. A LOT of documents. The largest customer of our system has as much as 130 million documents, and the amount keeps growing exponentially. With this in mind, we knew early that we had to shard up and cluster the Solr-collection, this meant using [SolrCloud](https://wiki.apache.org/solr/SolrCloud). SolrCloud uses [Apache ZooKeeper](https://zookeeper.apache.org/) to keep track of config files, live nodes, coordination etc. Zookeeper is a distributed file system that supports putting watchers on its files, thus calling back to the application when a file changes.

As a abstraction layer we use [SolrNet](https://github.com/mausch/SolrNet) to talk to Solr. The only problem here is that SolrNet has no support for SolrCloud ([yet!](https://github.com/mausch/SolrNet/pull/187)), which means that there is no good way to talk to the cluster, oure system's application server must connect to a single, specified node. If this node goes down, we lose the connection to the cluster. So we have redundancy in SolrCloud, but the consumer can't use it? We can't live with that.

###The solution
We decided to write a ZooKeeper-API ourself, letting the application server receive a suitable Solr-node on startup. If this node goes down in production, a watcher on the clusterstate-file kicks inn and changes the active Solr-node, giving us failover without restart.

There are some ways to talk to Zookeeper from .NET, mainly the [Apache ZooKeeper .NET Client](https://www.nuget.org/packages/ZooKeeper.Net/) package on NuGet. In this case, I decided to exploit the opportunity to test out [IKVM](http://www.ikvm.net/), a (magical) project that let's you convert Java JAR-files to DLLs. Yeah.

Here is some code from the API:


    using System;
    using System.Diagnostics.CodeAnalysis;
    using Common.Logging;
    using org.apache.zookeeper;
    using org.apache.zookeeper.data;

    public class ZookeeperNodeDataChangeWatcher : Watcher, org.apache.zookeeper.AsyncCallback.DataCallback,
        org.apache.zookeeper.AsyncCallback.StatCallback, IDisposable
    {
        private static readonly ILog s_log = LogManager.GetLogger(typeof(ZookeeperNodeDataChangeWatcher));

        private const int SessionTimeout = 2000;

        private static string s_nodePath;

        private readonly ZooKeeper m_zookeeper;

        private readonly IObserver<Maybe<byte[]>> m_observer;

        private object m_nodeExists;
        private bool IsDisposed { get; set; }

        public ZookeeperNodeDataChangeWatcher(string zookeeperName, IObserver<Maybe<byte[]>> observer, string nodePath)
        {
            this.m_observer = observer;
            this.IsDisposed = false;
            s_nodePath = nodePath;
            this.m_zookeeper = new ZooKeeper(zookeeperName, SessionTimeout, this);

            this.GetData();

            s_log.Info(string.Format("Zookeeperwatcher: started watcher: {0},{1}", this.m_zookeeper, s_nodePath));
        }

        private void GetData()
        {
            try
            {
                this.m_zookeeper.getData(s_nodePath, this, this, null);

            }
            catch (Exception e)
            {
                this.m_observer.OnError(e);
                s_log.Info(string.Format("Error in getData: {0},{1},{2}", this.m_zookeeper, s_nodePath, e.Message));
            }
        }

        private void WatchAgain()
        {
            try
            {
                this.m_zookeeper.exists(s_nodePath, this, this, null);
                s_log.Debug(string.Format("Rewatching: {0},{1}", this.m_zookeeper, s_nodePath));
            }
            catch (Exception e)
            {
                this.m_observer.OnError(e);
                s_log.Error(string.Format("Error in rewatching: {0},{1},{2}", this.m_zookeeper, s_nodePath, e.Message));
            }
        }

        public void processResult(int i, string str, object obj, byte[] barr, Stat s)
        {
            var returnCode = i;
            var data = barr;

            if (this.IsDisposed)
            {
                return;
            }

            if (returnCode == KeeperException.Code.OK.intValue())
            {
                this.m_nodeExists = true;
                this.m_observer.OnNext(Maybe.Return(data));
                this.WatchAgain();
            }
            else if (returnCode == KeeperException.Code.NONODE.intValue())
            {
                this.m_nodeExists = false;
                this.WatchAgain();
                this.m_observer.OnNext(Maybe.Empty<byte[]>());
            }
            else
            {
                this.m_observer.OnError(
                    KeeperException.create(KeeperException.Code.get(i)));
            }
        }

        public void processResult(int i, string str, object obj, Stat s)
        {
            if (this.IsDisposed)
            {
                return;
            }

            var returnCode = i;
            if (returnCode == KeeperException.Code.OK.intValue() ||
                returnCode == KeeperException.Code.NONODE.intValue() ||
                returnCode == KeeperException.Code.NODEEXISTS.intValue())
            {
                
                var nodeExistsNow = s != null;
                var oldExists = (bool?)this.m_nodeExists;
                this.m_nodeExists = nodeExistsNow;
                if (nodeExistsNow && nodeExistsNow != oldExists)
                {
                    this.GetData();
                }
            }
            else
            {
               this.m_observer.OnError(KeeperException.create(
                    KeeperException.Code.get(returnCode)));
            }
        }

        public void process(WatchedEvent e)
        {
            if (this.IsDisposed)
            {
                return;
            }

            if (s_nodePath != null && (e == null || s_nodePath != e.getPath()))
            {
                this.WatchAgain();
                return;
            }

            var type = e.getType();
            if (type == Watcher.Event.EventType.NodeCreated ||
                type == Watcher.Event.EventType.NodeDataChanged)
            {
                this.GetData();
            }
            else if (type == Watcher.Event.EventType.NodeDeleted)
            {
                this.m_nodeExists = false;
                this.m_observer.OnNext(Maybe.Empty<byte[]>());
                this.WatchAgain();
            }
            else
            {
                this.WatchAgain();
            }
        }

        public void Dispose()
        {
            s_log.Debug(string.Format("Watcher is disposed: {0},{1}", this.m_zookeeper, s_nodePath));
            GC.SuppressFinalize(this);
        }
    }

The implementation of the Clusterstate-watcher:


    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Reactive.Linq;
    using System.Text;
    using Common.Logging;
    using DIPS.Zookeeper.Model;

    using Newtonsoft.Json;
    using Newtonsoft.Json.Linq;

    public class ZooKeeperClusterstateWatcher
    {
        public event ActiveSolrCollectionChangedDelegate ActiveSolrCollectionChanged;

        private static readonly ILog s_log = LogManager.GetLogger(typeof(ZooKeeperClusterstateWatcher));

        private readonly string m_solrCollectionName;

        private readonly string m_zookeeperConnectionString;

        private const string ClusterState = "/clusterstate.json";

        public ZooKeeperClusterstateWatcher(string zookeeperConnectionString, string collectionName)
        {
            this.m_solrCollectionName = collectionName;
            m_zookeeperConnectionString = zookeeperConnectionString;
            Subscriber();
        }

        public IObservable<Maybe<byte[]>> WatchData(string zookeeper, string nodePath)
        {
            var ob = Observable.Create<Maybe<byte[]>>(
                    observer =>
                        new ZookeeperNodeDataChangeWatcher(zookeeper, observer, nodePath));
            return ob;
        }

        public void UpdateActiveSolrCollection(SolrCollection collection)
        {
            ActiveSolrCollection.SolrCollection = collection;
            if (ActiveSolrCollectionChanged != null)
            {
                ActiveSolrCollectionChanged(this, new ActiveSolrCollectionChangedDelegateArgs() { ActiveCollection = collection });
            }
        }

        public void HandleException(Exception ex)
        {
            s_log.Error(string.Format("Error in clusterstate.json watcher : {0}", ex));

            if (ex is org.apache.zookeeper.KeeperException.ConnectionLossException)
            {
                Subscriber();
            }
        }

        public SolrCollection ParseClusterstateJson(string content)
        {
            try
            {
                var clusterstate = JsonConvert.DeserializeObject<dynamic>(content);
                var collection = new SolrCollection { Name = this.m_solrCollectionName };
                var varShards = new List<Shard>();

                foreach (var solrCollection in clusterstate[collection.Name])
                {
                    collection.MaxShardsPerNode = int.Parse(clusterstate[collection.Name]["maxShardsPerNode"].ToString());
                    collection.ReplicationFactor = int.Parse(clusterstate[collection.Name]["replicationFactor"].ToString());
                    collection.AutoAddReplicas = bool.Parse(clusterstate[collection.Name]["autoAddReplicas"].ToString());
                    collection.Router = new Router { Name = clusterstate[collection.Name]["router"]["name"].ToString() };
                    foreach (JObject shards in solrCollection.Children<JObject>())
                    {
                        foreach (var shard in shards.Properties().Where(prop => !prop.Name.Equals("name")))
                        {
                            var varShard = new Shard();

                            if (clusterstate[collection.Name]["shards"][shard.Name]["state"].ToString().Equals("active"))
                            {
                                varShard.IsActive = true;
                            }

                            varShard.Name = shard.Name;
                            varShard.Range = clusterstate[collection.Name]["shards"][shard.Name]["range"].ToString();
                            var coreNodes = new List<CoreNode>();

                            foreach (var replicas in clusterstate[collection.Name]["shards"][shard.Name]["replicas"])
                            {
                                var coreNode = new CoreNode();
                                foreach (JObject replicass in replicas.Children<JObject>())
                                {
                                    foreach (var coreNodeInReplica in replicass.Properties())
                                    {
                                        switch (coreNodeInReplica.Name)
                                        {
                                            case "state":
                                                coreNode.State = coreNodeInReplica.Value.ToString();
                                                break;
                                            case "node_name":
                                                coreNode.NodeName = coreNodeInReplica.Value.ToString();
                                                break;
                                            case "core":
                                                coreNode.CoreName = coreNodeInReplica.Value.ToString();
                                                break;
                                            case "base_url":
                                                coreNode.BaseUrl = coreNodeInReplica.Value.ToString();
                                                break;
                                            case "leader":
                                                coreNode.IsLeader = Convert.ToBoolean(coreNodeInReplica.Value.ToString());
                                                break;
                                        }
                                    }

                                    coreNodes.Add(coreNode);
                                }

                                varShard.ReplicaCores = coreNodes;
                            }

                            varShards.Add(varShard);
                        }
                    }
                }

                collection.Shards = varShards;
                s_log.Debug(string.Format("Registered new Solr-collection: {0}. Number of shards is {1}", collection.Name, collection.Shards.Count()));
                return collection;
            }
            catch (Exception e)
            {
                s_log.Error(string.Format("Zookeeperwatcher: error in clusterstate.json parser : {0}", e.Message));
            }

            return new SolrCollection();
        }

        private void Subscriber()
        {
            var observableClusterState = this.WatchData(m_zookeeperConnectionString, ClusterState);
            var subscription =
                observableClusterState.Subscribe(
                    x => this.UpdateActiveSolrCollection(this.ParseClusterstateJson(Encoding.UTF8.GetString(x.Value))),
                    this.HandleException);
        }
    }

    public delegate void ActiveSolrCollectionChangedDelegate(object sender, ActiveSolrCollectionChangedDelegateArgs args);


From code we simply read out ``ActiveSolrCollection.SolrCollection.LeaderNode``.

>NOTE: In retrospect, we ended up rewriting and using the [Apache ZooKeeper .NET Client](https://www.nuget.org/packages/ZooKeeper.Net/) package instead, simply because IKVM makes much mess, giving us dependencies to a lot of DLLs and so on.

### Some slides
I ended up giving a short talk about the work as well as presenting extension that can be made. In the future I will look at ZooKeeper as a place to place distributed config for services, as well as service discovery.

<iframe src="//www.slideshare.net/slideshow/embed_code/46646103" width="425" height="355" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/mostii/zookeeper-aware" title="Zookeeper-aware application server" target="_blank">Zookeeper-aware application server</a> </strong> from <strong><a href="//www.slideshare.net/mostii" target="_blank">Andreas Mosti</a></strong> </div>

This work is inspired by the work of the guys over at [Palladium Consulting](http://www.palladiumconsulting.com/2014/06/using-zookeeper-from-net-ikvm-task/).
