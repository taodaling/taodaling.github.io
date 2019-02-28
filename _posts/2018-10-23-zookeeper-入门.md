---
categories: tool
layout: post
---
- table
{:toc}

# Introduction

ZooKeeper is a distributed, open-source coordination service for distributed applications. It exposes a simple set of primitives that distributed applications can build upon to implement higher level services for synchronization, configuration maintenance, and groups and naming.

# Programming

## Data Model

Zookeeper has a hierarchal name space, much like a distributed file system. The only difference is that each node in the namespace can have data associated with it as well as children. Paths to nodes are always expressed as a canonical, absolute, slash-separated paths. Any unicode character can be used in a path subject to the following constraints.

- \u0000-\u001F and \u007F-\u009F and \ud800-\uF8FF and \uFFF0-\uFFFF are forbiden
- You can use '.' or '..' as part of node name, but you can't use '.' or '..'  individually.
- The token 'zookeeper' is reserved.

### ZNodes

Every node in a zookeeper tree is referred to as a znode. Znodes maintain a stat structure that includes version number for data changes, acl changes. The stat structure also has timestamps. The version number, together with timestamp, allows zookeeper to validate the cache and to coordinate updates. Each time a znode's data changes, the version number increases. For instance, whenever a client retrieves data, it also receives the version of the data. And when a client performs an update or a delete, it must supply the version of the data of the znode it is changing. If the version it supplies doesn't match the actual version of the data, the update will fail. 

