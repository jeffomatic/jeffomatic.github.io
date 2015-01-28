---
layout: post
title: Redis as a mutex service
description: How to use Redis for distributed mutual exclusion.
---

<a href="https://www.flickr.com/photos/xserve/368758286" title="Lock by Lok Leung, on Flickr"><img src="https://farm1.staticflickr.com/115/368758286_e4dcb5ae53.jpg" width="500" height="334" alt="Lock"></a>

### TL;DR

Because Redis has atomic mutators that return useful information on what got modified, you can use it to provide [test-and-set](http://en.wikipedia.org/wiki/Test-and-set), the essential primitive operation for mutual exclusion. This means you can synchronize two or more independent processes, even if they are running on different machines, as long as they can both talk to the same Redis server.

### WTF

I strongly suspect that the following is an insane, hacky workaround for a problem that most likely has a more robust and formal solution. But since I did not discover any such solution after ten minutes of looking, this is what I did.

### The problem domain

Suppose you have an application with a rather complex process model. Suppose it has a couple of processes running in independent execution contexts, let's say two web requests that may or may not be running on the same server. Also suppose that both of these processes are likely to attempt the exact same Expensive Operation, let's say retrieving a resource from a REST API off in a distant corner of the internet.

Now suppose you would like to cache the result of your Expensive Operation. If any process were to discover that the Expensive Operation had not yet been cached, you'd certainly prefer that only one process attempt to execute it. In other words, you'd like your processes to:

- Look into the cache
- If nothing's in the cache, reserve the right to perform the Expensive Operation via some locking mechanism
- Once the lock is acquired, perform the Expensive Operation and store the result in the cache
- Relinquish the right to perform the Expensive Operation
- Now: where exactly does "some locking mechanism" come from? For threads running in the same process, you get this by giving each thread a handle to the same mutex. For processes running on the same file system, you might achieve this through [file locking](http://en.wikipedia.org/wiki/File_locking).

But let us further suppose that your processes are running on Heroku or some other kind of virtualized host, and thus don't have reliable, persistent access to a common file system. In this case, we'd need some network-accessible resource that our processes can use as a common repository of locks.

<a href="https://www.flickr.com/photos/psychic_sometimes/3851533870" title="Commodore VIC 20 by ju4n, on Flickr"><img src="https://farm3.staticflickr.com/2583/3851533870_c121e2f6fb.jpg" width="500" height="375" alt="Commodore VIC 20"></a>

### test-and-set

In theory, it doesn't really matter what our lock-providing resource actually is, as long as it can be repurposed to provide atomic [test-and-set](http://en.wikipedia.org/wiki/Test-and-set) operations that are namespaced along on arbitrary string handles.

In other words, if our handle is `foobar`, our lock provider should be able to receive commands along the lines of "set the variable `foobar` to 1 if and only if `foobar` isn't already 1, and return the value of `foobar`", or perhaps "insert the string `foobar` into a list, but only if it's not already in there, and also tell me whether or not it was already in there to begin with." What's more is that this type of command, verbose and chunky as it is, needs to happen atomically, i.e., it cannot be interrupted even if other similar commands are received during its execution.

For the classical operating system, this is reducible to some blackbox operation provided by the CPU. When we can't assume the presence of a unique CPU, we'll need some other unique resource that our processes can query.

<a href="https://www.flickr.com/photos/tbuser/5577325006" title="Database Diagram FAIL by Tony Buser, on Flickr"><img src="https://farm6.staticflickr.com/5026/5577325006_373a3267b0.jpg" width="500" height="316" alt="Database Diagram FAIL"></a>

### The guts

Mostly because it was immediately available, I decided to use [Redis](http://redis.io/) as my lock provider.

For those unfamiliar, Redis is a piece of server software that provides a fast, network-accessible key/value store. One of Redis's nice features is that it can store complex data structures like associative arrays and sets. Another nice feature is that it retains its dataset entirely in memory. This means that external requests won't directly trigger disk operations, and the resulting lack of I/O blocking means that these requests can be served efficiently by a single thread.

What all of this niceness buys you, in turn, is *atomic set insertions*. In other words, you can add to an array stored in Redis and make sure that the element you're adding 1) only exists once in the array (a property of all set operations), and 2) happens without getting interrupted by other requests (a property of single-threaded processes).

With this kind of firepower, all you'd need in order to emulate test-and-set is to know whether your set insertion actually added anything new to the set. And this is in fact the exact behavior of [`sadd`](http://redis.io/commands/sadd), Redis's command for set insertion. `sadd` takes two arguments: the name of the set you're adding to, and one or more elements you want to insert into the set, and it returns the difference of 1) the size of the set after the insert and 2) the size of the set before the insert. Since one or more of the elements you tried to insert may already have been in the set, `sadd`'s return value isn't always the same as the number of elements you were trying to insert.

Thus, our recipe for locking in Redis:

- Insert a lock handle into our set of locks: `sadd set_of_all_named_locks, lock_handle`
- If the return value is 0, sleep the current process a little, and re-attempt the above.
- If the return value is 1, then our current process has acquired the lock for `lock_handle`. Once we no longer need the lock, we just have to remove `lock_handle` from the set of locks using [`srem`](http://redis.io/commands/srem).

Here's an example. If you don't know Ruby, hopefully the comments can provide hints as to what is going on:

{% highlight ruby %}
# CAVEAT EMPTOR: This is excerpted code that I pulled out of context,
# re-arranged, renamed the variables of, and otherwise dicked around with
# a lot without having put it back into a Ruby interpreter, so it's
# possible that there's some gnarly typo or bug in here. In short, don't
# cut-and-paste this code.

class AlreadyLockedException < Exception; end

def redis_spinlock(redis, lock_handle, opts = {}, &critical_section)
  lock_set = opts[:lock_set] || 'named_locks'
  sleep_time = opts[:sleep] || 0.1
  spins = 0

  begin
    # Attempt to acquire the lock by adding our lock handle to the set of
    # locks. If the number of elements added to the set isn't 1, then
    # another process has already acquired the lock.
    #
    # Syntax subtlety: in somewhat obsequious fashion, the Ruby Redis
    # client returns bool(true) rather than int(1) if the element was
    # successfully inserted and was not passed as a single-element
    # array.
    added = redis.sadd lock_set, lock_handle
    raise AlreadyLockedException unless added === 1 || added === true

    begin
      # Here, we'll perform the operation that originally required the
      # lock. We'll pass the number of times we had to spin on the lock
      # to the received block of code. In most cases, you'll probably
      # only care whether the spin count was zero or non-zero, i.e.,
      # whether you actually had to compete with another process for the
      # lock.
      critical_section.call(spins)
    ensure
      # Unlock, even if we run into errors
      redis.srem lock_set, lock_handle
    end
  rescue AlreadyLockedException
    spins += 1
    sleep(sleep_time) if sleep_time > 0
    retry
  end
end
{% endhighlight %}

You might implement the above caching scheme as follows:

{% highlight ruby %}
# In our example, "cache" is some resource accessible by all of our
# processes and provides a run-time interface similar to Ruby's Hash
# class.
def cache_lookup(redis, resource_handle, cache, &run_on_cache_miss)
  begin
    cache.fetch(resource_handle)
  rescue KeyError
    redis_spinlock(redis, resource_handle) do |spin_count|
      if spin_count > 0
        # We had to wait before acquiring the lock, which means something
        # probably is in the cache now. We should check the cache again.
        begin
          return cache.fetch(resource_handle)
        rescue KeyError
          # do nothing
        end

      cache[resource_handle] = run_on_cache_miss.call
    end
  end
end
{% endhighlight %}

An alternative method that might be a smidge faster, which I didn't think of until just now
Actually, every operation in Redis is atomic, so we could use increments and decrements ([`incr`](http://redis.io/commands/incr) and [`decr`](http://redis.io/commands/decr)) to provide a more classical-looking test-and-set:

- Increment the variable corresponding to our lock handle: `incr lock_handle`
- If the return value of incr is 1, then we've acquired the lock. Perform the operation.
Regardless of the return value of `incr`, decrement the value corresponding to our lock handle: `decr lock_handle`
- If the return value of `incr` was 0, we should try incrementing again.

### Regrets

It occurs to me that in a world where the file system is routinely repurposed to provide cross-process locking, it's not so far fetched to use Redis as a lock system for processes that can only communicate with one another via TCP/IP. Still, it feels a little as if I've brought a shotgun to a knife fight. It would be great if there was a network-enabled lock server that responded to named lock requests in a similar fashion, only purpose-built and with a lighter interface. It sure seems like it would be a handy thing to have from time to time. In fact, I'd be kind of surprised if someone hadn't already come up with something like this, in like 1982 or something.

### Addenda (2012.5.3)

- The term that I should have searched for in the first place is either "distributed mutex" or "distributed synchronization".
- I was off by at least a few years. Greybeard academia has been contending with this issue since before 1978. cf. [Lamport](http://en.wikipedia.org/wiki/Lamport%27s_Distributed_Mutual_Exclusion_Algorithm) (1978-ish) and [Ricart-Agrawala](http://en.wikipedia.org/wiki/Ricart%E2%80%93Agrawala_algorithm) (1981).
- Helpful commenters have pointed me to [Apache ZooKeeper](http://zookeeper.apache.org/).
- Naturally, I was not nearly the first person to think of using Redis in this manner; the first was probably someone on the Redis team itself, since they seem to have implemented the [`setnx`](http://redis.io/commands/setnx) instruction for exactly this purpose. And at least a [few](https://github.com/0ctave/node-mutex) [libraries](https://github.com/tanin47/ruby_redis_lock) [exist](https://github.com/dv/redis-semaphore) to take advantage of this.