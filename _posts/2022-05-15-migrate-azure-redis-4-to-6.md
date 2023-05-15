---
title: "Azure Redis Cache. Migrating from v4 to v6"
categories:
  - Blog
tags:
  - Azure
  - Caching
  - Redis
---

# Azure Redis Cache v4 to v6 upgrade by June 30th, 2023

As you probably know, Azure have the the Redis Cache service upgrade planned on June 30th, 2023
[Microsoft.com Post](https://azure.microsoft.com/en-us/updates/upgrade-your-azure-cache-for-redis-instances-to-use-redis-version-6-by-30-june-2023/)
https://azure.microsoft.com/en-us/updates/upgrade-your-azure-cache-for-redis-instances-to-use-redis-version-6-by-30-june-2023/

In this article I'll summarize some points for you about this process if you are also using Azure Redis Cache v4 in your application, as I do in several projects for a company that I work for.

### Manual vs Automatic migration
The official documentation and my communication with the MS Azure support team say that you can either migrate your instance manually by following the instruction steps on the link above or do nothing. The second option is a bit more interesting, since in this case the exact moment of upgrading (after the June 30th, 2023) is not determined and it will happen even if you have "Scheduled updates" setting completely disabled.

### What will happen with the data?
On any plan starting with C1 Standard, according to the MS support, the data will be persisted. That is because of having more than one replica nodes on such plans where the primary node could be changed while the others are upgrading and rebooting.
On the basic tier C0 - the data will be lost.

### How to keep the data on the C0 plan?
The only way of keeping data on this plan is creating a backup, copying the data, upgrade an instance, and copying it back.
As the Microsoft recommends, you can use
[[redis-copy](https://github.com/deepakverma/redis-copy/releases/download/alpha/Release.zip)] the command line tool to copy all keys with their properties to a new/backup instance.
The command line for running this tool will be pretty straightforward: a source instance connection string, a source instance authorization data, and the same for the destination. Unfortunately, the tool does not inform about the progress of the copying operation, it only shows 100% when the process is done. So, if you want to monitor the process and finally check the results, it is better to open the Metrics tabs of a corresponding instance in Azure and check that the metric of "Total keys" is increasing and eventually become the same as it it in the source instance.

### Stackexchange Redis library versions
If a project that you are working on is using Stackexchange library for communicating with a Azure Redis cache instance and its version i Stackexchange.Redis.Strongname, 1.2.6, probably that is also a suitable moment to upgrade it to the latest one Stackechange.Redis 2.6.111. Please check the[Stackexchange Changelog](https://stackexchange.github.io/StackExchange.Redis/ReleaseNotes.html), since the version 2.0.495 had the breaking changes.

### Microsoft.Web.Redis.RedisSessionStateProvider
According to the nuget feed and the github activity this provider got the new version in March 2023, the previous update of this nuget package happened in 2018. So, the new 5.0.0 version of Microsoft.Web.Redis.RedisSessionStateProvider is using the latest Stackexchange.Redis library (2.6.101) and has the breaking change in the way of keeping the ASP.NET session data: keys were renamed and the way of serializing session data was changed as well (from the Redis hashset with the serialized values to the purely binary serialization of the whole ISessionStateItemCollection). It means that you cannot just upgrade package and having your ASP.NET session's data available again.

#### Migrating to the latest Microsoft.Web.Redis.RedisSessionStateProvider with keeping the ASP.NET session data
The link to the commit where the keys renamed [Keys renaming commit](https://github.com/Azure/aspnet-redis-providers/commit/790f764780e48dc5042ee8e89f9749ea3994a136)
```
{{app}_{id}}_Data -> {{app}_{id}}_SessionStateItemCollection
{{app}_{id}}_Write_Lock -> {{app}_{id}}_WriteLock
{{app}_{id}}_Internal -> {{app}_{id}}_SessionTimeout
```
Moreover, the types (and as a result Redis command to operate by them) were changed:

{% include figure image_path="/assets/images/posts/2022-05-15-migrate-azure-redis-4-to-6/redisstateprovider-diffs-in-lua-scripts-and-serialization.png" alt="Differences in the way of serialization and LUA scripts" caption="" %}

So, in order to be able to read "old" keys and data the new version/fork of the existing RedisSessionStateProvider should be created where:
- Old way of creating keys will be used for generating them and checking such keys in lua script
- If the old session keys exist, deserialize the data in old format and generate output in the new one
- Properly expire/clean up old keys
- I'd skip dealing here with the write locks, since data will be stored in the new format anyway, and the write lock in the old format does not make much sense here

#### Forked version of Microsoft.Web.Redis.RedisSessionStateProvider with the proposed changes 
You can build the forked version of the RedisSessionStateProvider with the ability of reading session state data in the old format that was implemnted by me:
[[Commit with the smooth migration to RedisSessionStateProvider that supports session state in the old format](https://github.com/Azure/aspnet-redis-providers/compare/main...sergeyfsv:aspnet-redis-providers-migration:feature/migration-to-500)]
