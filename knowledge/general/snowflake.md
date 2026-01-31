# Snowflake ID

## Overview

Snowflake IDs are 64-bit unique identifiers designed for distributed systems, created by Twitter (now X) in 2010. The name comes from the belief that every snowflake has a unique structure.

## Format

A Snowflake ID is 64 bits (63 bits used to fit in a signed integer):

- **41 bits**: Timestamp in milliseconds since epoch (Twitter uses epoch: 1288834974657)
- **10 bits**: Machine/worker ID (prevents collisions across machines)
- **12 bits**: Sequence number (allows multiple IDs per millisecond on same machine)

This structure makes Snowflake IDs:
- **Sortable by time** - IDs created later have higher values
- **Extractable timestamp** - creation time can be derived from the ID
- **Globally unique** - machine ID + sequence prevents collisions

## Why Snowflake IDs Are Needed

### Requirements for Distributed System IDs

1. **Uniqueness** - No collisions across multiple machines/datacenters
2. **Sortability** - Order by creation time without querying timestamps
3. **Time extraction** - Derive creation time from ID itself
4. **High throughput** - Generate millions of IDs per second
5. **No coordination** - Machines generate IDs independently

### Why Other Approaches Fail

#### Auto-incrementing Integers
- **Problem**: Requires centralized database or coordination
- **Failure**: Single point of failure, bottleneck, doesn't scale
- **Use case**: Single-server applications only

#### UUIDs (v1-v4)
- **v1 (MAC address + timestamp)**: 
  - Privacy concerns (exposes MAC address)
  - Not sortable (timestamp not in most significant bits)
- **v4 (random)**:
  - Not sortable at all
  - Requires database indexes for time-based queries
- **Use case**: When uniqueness is enough, sortability not required

#### Database Sequences
- **Problem**: Requires database round-trip for each ID
- **Failure**: Latency bottleneck, doesn't scale horizontally
- **Use case**: Low-throughput, centralized systems

#### Timestamp + Random
- **Problem**: Collision risk in same millisecond
- **Failure**: No machine/sequence component to prevent duplicates
- **Use case**: Low-throughput systems where collisions are acceptable

## Other Approaches and When They're Used

### UUIDv7
- **Format**: 48-bit timestamp + random bits
- **Pros**: Sortable, time-extractable, standard format
- **Cons**: Larger than Snowflake (128 bits vs 64 bits)
- **Use case**: When standard format is preferred, size not critical

### Instagram's Modified Snowflake
- **Format**: 41-bit timestamp + 13-bit shard ID + 10-bit sequence
- **Difference**: Larger shard ID field (13 vs 10 bits)
- **Use case**: Systems needing more shards/machines

### Mastodon's Format
- **Format**: 48-bit millisecond timestamp + 16-bit sequence
- **Difference**: No machine ID, uses UNIX epoch
- **Use case**: Simpler systems with fewer machines

### Database-Generated IDs
- **Examples**: PostgreSQL SERIAL, MySQL AUTO_INCREMENT
- **Use case**: Single-database applications, low scale

### KSUID (K-Sortable Unique Identifier)
- **Format**: 32-bit timestamp + 128-bit random payload
- **Pros**: Sortable, URL-safe base62 encoding
- **Cons**: Larger than Snowflake
- **Use case**: When human-readable, URL-safe IDs are needed

## Key Advantages of Snowflake IDs

1. **Time-ordered**: Natural sorting by creation time
2. **Compact**: 64 bits fits in most integer types
3. **Decentralized**: No coordination needed between machines
4. **High throughput**: Can generate thousands per millisecond per machine
5. **Time extraction**: Can derive creation time without storing it separately

## Implementation Considerations

- **Machine ID allocation**: Must ensure unique IDs across machines (often split into datacenter ID + worker ID)
- **Clock synchronization**: Requires synchronized clocks (NTP typically sufficient)
- **Epoch selection**: Choose epoch based on expected system lifetime
- **Sequence overflow**: Handle case when sequence exceeds 12 bits in same millisecond (wait for next millisecond)

## References

- Twitter's original announcement: https://blog.x.com/engineering/en_us/a/2010/announcing-snowflake
- Used by: X/Twitter, Discord, Instagram (modified), Mastodon (modified)
