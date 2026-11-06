# Etcd::Lock

Ruby gem for coordinating exclusive jobs using etcd's atomic compare-and-swap feature.

https://coreos.com/etcd/docs/latest/api.html#atomic-compare-and-swap

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'etcd-lock'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install etcd-lock

## Usage

```ruby
# Initialize a new coordinator for my application
elc = Etcd::Lock::Coordinator.new('my-app')

# Perform some work, using a lock that lives for 30 seconds to
# ensure the job with not run anywhere else in that time period
elc.run('my-job', ttl: 30) do
  perform_some_work
end
```

If `ttl` is not specified, a default of _10 seconds_ will be used.

Be default, the lock will live until the ttl expires, no matter how
long the supplied block takes to run. If you want the lock to expire
when the job is finished, use the `remove: true` parameter:

```ruby
# Force the lock to expire as soon as the block has been executed
elc.run('my-job', ttl: 3600, remove: true) do
  perform_some_work
end
```

Note: if the block raises an uncaught exception, the lock will persist
until the ttl expires, even when `remove: true` is specified.
