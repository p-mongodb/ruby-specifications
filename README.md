# Move Query Cache to Driver 
Author: Grace Chong 

Reviewer: Emily Giurleo

## Summary 
The main goal of this project is to reimplement Mongoid Query Cache as part of the Ruby Driver, so that eventually Mongoid can adapt the new Query Cache implementation without having dependencies on the Driver code. 

## Motivation
The Mongoid Query Cache allows users to cache the results of find queries, preventing the ODM from making multiple network round-trips to the database when a user executes the same query multiple times. However, the current implementation of the query cache is flawed:
* It relies heavily on Ruby Driver internals, creating a dangerous coupling where a change to the Driver could inadvertently break the Query Cache.
* It relies on Driver legacy read retries, which are outdated. This dependency prevents the removal of legacy read retries from the Driver, making it harder to reason about retries in the Driver.
* It contains copy-and-pasted Driver cursor code, most of which is now outdated.

This project would make both Mongoid and the Ruby Driver easier to maintain and change by introducing modularity and separation of concerns to the Query Cache code.

## Current Behavior 
Currently, the Cursor implemented by the driver does not support caching.

QueryCache as implemented in Mongoid is a cache of database queries that operate on a per-request basis. It caches queries as hashes on the current thread that is being executed. QueryCache invalidates or clears out the query cache on certain operations, such as all write operations, that can modify the query results and thus invalidate the stored cache. 

More specifically, Mongoid’s QueryCache caches a Cursor known as CachedCursor, which attempts to load documents from memory first before going back to the database if the same query has already been executed. It inherits from the Driver's version of the Cursor, which stores information such the CollectionView defining the query, the result of the first execution of the query, the server the cursor is locked to, and other options.  

## New Behavior

There shouldn’t be any functional differences between the implementation of caching in the driver compared to caching in Mongoid, so behaviorally they should act the same way. The way a user would interface with the new caching feature in the Driver cursor should be the same as in Mongoid. 

Here are the following proposed features: 
* Implement a caching cursor in the Ruby Driver.
* Implement an API in the Ruby Driver that allows the user to enable and disable the query cache on a global level and within the scope of a block.
* Update Mongoid to use the new Ruby Query Cache implementation.

## API Examples

In general, the way the user would interface with the QueryCache would not change once it is reimplemented in Driver. 
Much like Mongoid, the Ruby Driver will allow the user to globally enable or disable the query cache.

```ruby
Mongo::QueryCache.enabled = true

It will also provide a block API that enables/disables the Query Cache within the context of the block:

Mongo::QueryCache.cache do
  # all queries within this block are cached
end
```

## Design

Here are some of the changes I plan on implementing to the driver in order to support a query caching feature:

* Add a QueryCache module to the driver which supports Cursor objects that have caching capabilities
* Enable toggling for QueryCache so that caching is only done if the ruby query cache is turned on. If the option is not enabled, the Cursor should work normally in the same way it did before.  
* Support storing query caches on locally executed threads as done in Mongoid
* Include query cache behavior for Collection class by making any write operations aliases to the query cache clear method
* Include query cache behavior for View classes by making any write operations aliases to the query cache clear method and adding functionality to retrieve cached Cursors from a cache_table with the appropriate keys
* Make changes to the each function in the Iterable module to handle if the cursor has be cached or not
* Modifying the way that the cache_key is maintained by adding logic so that queries with different limits can use the same cachedCursor if possible (ex. A query to retrieve the first 100 documents can encompass a query to retrieve the first 10)
* Using modern retry read methods currently supported by the Driver so when Mongoid gets updated to reflect the new Query Cache, the legacy read retries can be removed.
* Add methods to the Cursor class to get and iterate over the cached queries, clear the query cache in the case where write operations are performed, and check whether the Query Cache has been enabled
* Store the output of the get_more function, which returns the batch of documents from a cursor
* Implement the user-facing caching API in driver for enabling, clearing and retrieving QueryCaches for methods found in Mongoid 

## Assumptions and risks

The biggest risk of this project is that any work done to the Ruby Driver will break the existing Query Cache implementation in Mongoid. 

## Testing Plan

Here are several areas that would require testing in order to ensure that no Mongoid Query Cache features are affected by the modifications to the Driver.

* The ruby driver cursor only does caching if the ruby query cache is turned on. If not, the cursor behaves the same as it’s always behaved. 
* A Cursor loads documents from memory if the same or similar query has been executed before
* Cursors are being cached appropriately with a cache_key 
* Query Caching feature does not use legacy read_with_retry methods
* Query Cache is invalidated and cleared if any write operations are performed 
* The current thread correctly stores the cached queries
* User can enable or disable query caching on the driver

## Documentation Plan

Documentation should be added for the following modifications:
* New QueryCache module and methods for getting the cached queries, clearing the query cache, and toggling the enabled query cache option
* View and Collection classes
* Iterable class
* Existing methods to support query caching behavior 
* Current Cursor in the driver 

## Implementation Plan/Schedule


| Description / JIRA ticket | Time  |
| :-----: | :-: |
| Implement Cached Cursor in Ruby Driver | 3 weeks |
| Implement user-facing caching API in Ruby Driver | 2 weeks |
| Replace Mongoid Query Cache with Driver Query Cache | 1 weeks |




