---
categories: tool
layout: post
---



{:toc}



# Introduction

ZooKeeper is a distributed, open-source coordination service for distributed applications. It exposes a simple set of primitives that distributed applications can build upon to implement higher level services for synchronization, configuration maintenance, and groups and naming.

# Programming

## Data Model

Zookeeper has a hierarchal name space, much like a distributed file system. The only difference is that each node in the namespace can have data associated with it as well as children. Paths to nodes are always expressed as a canonical, absolute, slash-separated paths. Any unicode character can be used in a path subject to the following constraints.

