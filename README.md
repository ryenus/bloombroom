# Bloombroom v1.2.0

[![build status](https://secure.travis-ci.org/colinsurprenant/bloombroom.png)](http://travis-ci.org/colinsurprenant/bloombroom)

- Standard **Bloomfilter** class for bounded key space
- **ContinuousBloomfilter** class for unbounded keys (**stream**)
- **Bitfield** class for compact & fast manipulation of bits field of arbitrary lenght 
- **BitBucketField** class for compact & fast manipulation of series of bit "buckets" of arbitrary size
- Fast **FNV hashing** using C implementation with FFI bindings.

The [Bloom filter](http://en.wikipedia.org/wiki/Bloom_filter) is a space-efficient probabilistic data structure that is used to test whether an element is a member of a set. False positives are possible, but false negatives are not.

Bloom filters are normally used in the context of a bounded set since the filter size must be known in advance. for a given filter capacity, its total bit size will affect the false positive error rate. The total number of bits required for a given filter can be computed from the required filter capacity and target error rate. See the [references](#references) section for more info.

This implementation uses [FNV hashing](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function). FNV is designed to be fast to compute.

### ContinuousBloomfilter
The [ContinuousBloomfilter](http://colinsurprenant.com/blog/2012/05/12/continuous-bloom-filter/) provides a bloom filter implementation which support unbounded stream of elements. Elements are expired after a chosen TTL. At initialization the filter capacity must be estimated for the numbers of elements expected over the given TTL period. 

For example to do dedupping on a stream with a rate of **5000 items/sec** over a period of **60 minutes** would require a filter capacity of **18M** elements. For a required error rate of **0.1%** the filter would need **246mb** of memory (which include all Ruby objects overhead).

The ContinuousBloomfilter uses 4 bits for each filter *position* or *bucket* (instead of 1 bit in a normal bloom filter) for keeping track of the keys TTL. 
The internal timer resolution is set to half of the required TTL (resolution divisor of 2). using 4 bits gives us
15 usable time slots (slot 0 is for the unset state). Basically the internal time bookeeping is similar to a
ring buffer using the current timer tick modulo 15. The timer ticks will be time slot=1, 2, ... 15, 1, 2 and so on. The total 
time of our internal clock will thus be 15 * (TTL / 2). We keep track of TTL by writing the current time slot 
in the key k buckets when inserted in the filter. For a key lookup if the interval betweem the current time slot and any of the k buckets value 
is greater than 2 (resolution divisor) we know this key is expired. See my [Continuous Bloom filter](http://colinsurprenant.com/blog/2012/05/12/continuous-bloom-filter/) blog post about this.

This means that an element is garanteed to not be expired before the given TTL but in the worst case could survive until 3 * (TTL / 2). 

### Hashing
Bloom filters require the use of multiple (k) hash functions for each inserted element. We actually simulate multiple hash functions by having just two hash functions which are actually the upper and lower 32 bits of our fast FFI FNV1a 64 bits hash function. Double hashing with one hash function. Very very fast. See [bloom_helper.rb](https://github.com/colinsurprenant/bloombroom/blob/master/lib/bloombroom/filter/bloom_helper.rb) and the [references](#references) section for more info on this technique.

## Installation

Tested on **OSX 10.8.2** and **Linux 12.10** with
- MRI Ruby 1.9.2, 1.9.3, 2.0.0
- JRuby 1.7.2, 1.7.3

Add this line to your application's Gemfile:
```ruby
gem 'bloombroom'
```
And then execute:
```sh
$ bundle
```
Or install it yourself as:
```sh
$ gem install bloombroom
```

## Usage

### Standard Bloom filter
``` ruby
require 'bloombroom'

bf = Bloombroom::BloomFilter.new(1000, 3) # 1000 bits and 3 hash functions

bf.add("key1")
bf.add("key2")

bf.include?("key1") # => true
bf.include?("key3") # => false
```

``` ruby
require 'bloombroom'

# compute optimal m,k for a filter capacity of 1000 elements and 0.1% error rate
m, k = Bloombroom::BloomHelper.find_m_k(1000, 0.001)

bf = Bloombroom::BloomFilter.new(m, k)

bf << "key1"
bf << "key2"

bf["key1"] # => true
bf["key3"] # => false
```
### Continuous Bloom filter

``` ruby
require 'bloombroom'

# 1000 buckets, 3 hash functions and a TTL of 2 seconds
bf = Bloombroom::ContinuousBloomFilter.new(1000, 3, 2)
bf.start_timer

bf << "key1"
bf << "key2"

bf["key1"] # => true
bf["key2"] # => true
bf["key3"] # => false

sleep(3)

bf["key1"] # => false
bf["key2"] # => false
bf["key3"] # => false
```

### FNV Hashing

``` ruby
Bloombroom::FNVFFI.fnv1_32("test")   # => 3157003241
Bloombroom::FNVFFI.fnv1a_32("test")  # => 2949673445
Bloombroom::FNVFFI.fnv1_64("test")   # => 10090666253179731817
Bloombroom::FNVFFI.fnv1a_64("test")  # => 18007334074686647077
```

### Bit Field

``` ruby
bf = Bloombroom::BitField.new(7)

bf.size            # => 7

bf[1] = 1          # set using array notation
bf.set(5)          # set using method
bf.total_set       # => 2

bf.to_s            # => "0100010"

bf.include?(5)     # => true
bf.zero?(5)        # => false

bf[1] = 0          # unset using array notation
bf.unset(5)        # unset using method

bf[2] = 1
bf.map{|bit| bit.zero?}  # => [true, true, false, true, true, true, true]
```

### Bit Bucket Field

``` ruby
bf = Bloombroom::BitBucketField.new(4, 3)   # 3 x buckets of 4 bits

bf[0] = 2
bf[0]         # => 2
bf.to_s       # => "0010 0000 0000"
bf.to_s(10)   # => "2 0 0"

bf.set(1, 15)
bf.get(1)     # => 15
bf.to_s       # => "0010 1111 0000"
bf.to_s(10)   # =>  "2 15 0"

bf.zero?(1)   # => false

bf.total_set  # => 2

bf.map{|bucket| bucket.zero?}  # => [false, false, true]
```

## Memory footprint
The calculated memory footprints **includes all Ruby objects overhead**. In fact the footprint is calculated by querying the process size (rss) before and after the bloom filter object initialization.

### Bloomfilter
``` sh
ruby benchmark/bloom_filter_memory.rb auto 1000000 0.01
ruby benchmark/bloom_filter_memory.rb auto 100000000 0.01
ruby benchmark/bloom_filter_memory.rb auto 100000000 0.001
```

- **1.0%** error rate for **1M** keys: **2.3mb**
- **1.0%** error rate for **100M** keys: **228mb**
- **0.1%** error rate for **100M** keys: **342mb**

### ContinuousBloomfilter
``` sh
ruby benchmark/continuous_bloom_filter_memory.rb auto 1000000 0.01
ruby benchmark/continuous_bloom_filter_memory.rb auto 100000000 0.01
ruby benchmark/continuous_bloom_filter_memory.rb auto 100000000 0.001
```

- **1.0%** error rate for **1M** keys: **9.1mb**
- **1.0%** error rate for **100M** keys: **914mb**
- **0.1%** error rate for **100M** keys: **1371mb**

## Simulation
This is an input stream simulation into the ContinuousBloomfilter. First a series to 32 x 20k random unique insertion keys & 20k random unique test keys not part of the insertion set are generated. At each iteration, 20k insertion keys are added, and 20k test keys are checked for inclusion and the internal timer tick is incremented. Since the life of our keys is of 3 timer ticks we chose a filter capacity of 3 x 20k elements. Specific m and k parameter will be computed for an error rate of 0.1% and 3 x 20k capacity. 

We see that as we add more keys, the test keys false positive rate is stable at the required error rate. In the second section, the same sequence is applied to a standard Bloomfilter to show that, obviously, the error rate will increase as more elements are added past the required capacity.

``` sh
ruby benchmark/continuous_bloom_filter_stats.rb 
```

```
generating lots of random keys

Continuous BloomFilter with capacity=60000, error=0.001(0.1%) -> m=862656, k=10
added 20000 keys, tested 20000 keys, FPs=0/20000 (0.000)%
added 20000 keys, tested 20000 keys, FPs=1/20000 (0.005)%
added 20000 keys, tested 20000 keys, FPs=17/20000 (0.085)%
added 20000 keys, tested 20000 keys, FPs=20/20000 (0.100)%
added 20000 keys, tested 20000 keys, FPs=23/20000 (0.115)%
added 20000 keys, tested 20000 keys, FPs=22/20000 (0.110)%
added 20000 keys, tested 20000 keys, FPs=22/20000 (0.110)%
added 20000 keys, tested 20000 keys, FPs=17/20000 (0.085)%
added 20000 keys, tested 20000 keys, FPs=18/20000 (0.090)%
added 20000 keys, tested 20000 keys, FPs=21/20000 (0.105)%
added 20000 keys, tested 20000 keys, FPs=11/20000 (0.055)%
added 20000 keys, tested 20000 keys, FPs=17/20000 (0.085)%
added 20000 keys, tested 20000 keys, FPs=18/20000 (0.090)%
added 20000 keys, tested 20000 keys, FPs=19/20000 (0.095)%
added 20000 keys, tested 20000 keys, FPs=21/20000 (0.105)%
added 20000 keys, tested 20000 keys, FPs=20/20000 (0.100)%
added 20000 keys, tested 20000 keys, FPs=24/20000 (0.120)%
added 20000 keys, tested 20000 keys, FPs=21/20000 (0.105)%
added 20000 keys, tested 20000 keys, FPs=22/20000 (0.110)%
added 20000 keys, tested 20000 keys, FPs=24/20000 (0.120)%
added 20000 keys, tested 20000 keys, FPs=15/20000 (0.075)%
added 20000 keys, tested 20000 keys, FPs=16/20000 (0.080)%
added 20000 keys, tested 20000 keys, FPs=16/20000 (0.080)%
added 20000 keys, tested 20000 keys, FPs=17/20000 (0.085)%
added 20000 keys, tested 20000 keys, FPs=22/20000 (0.110)%
added 20000 keys, tested 20000 keys, FPs=21/20000 (0.105)%
added 20000 keys, tested 20000 keys, FPs=24/20000 (0.120)%
added 20000 keys, tested 20000 keys, FPs=16/20000 (0.080)%
added 20000 keys, tested 20000 keys, FPs=17/20000 (0.085)%
added 20000 keys, tested 20000 keys, FPs=24/20000 (0.120)%
added 20000 keys, tested 20000 keys, FPs=17/20000 (0.085)%
added 20000 keys, tested 20000 keys, FPs=19/20000 (0.095)%
Continuous BloomFilter 640000 adds + 640000 tests in 16.95s, 75537 ops/s

BloomFilter with capacity=60000, error=0.001(0.1%) -> m=862656, k=10
added 20000 keys, tested 20000 keys, FPs=0/20000 (0.000)%
added 20000 keys, tested 20000 keys, FPs=1/20000 (0.005)%
added 20000 keys, tested 20000 keys, FPs=17/20000 (0.085)%
added 20000 keys, tested 20000 keys, FPs=131/20000 (0.655)%
added 20000 keys, tested 20000 keys, FPs=453/20000 (2.265)%
added 20000 keys, tested 20000 keys, FPs=1162/20000 (5.810)%
BloomFilter 120000 adds + 120000 tests in 1.64s, 146008 ops/s
```

## Benchmarks
All benchmarks have been run on a MacbookPro with a 2.66GHz i7 with 8GB RAM on OSX 10.6.8 with MRI Ruby 1.9.3p194

### Bloomfilter

``` sh
ruby benchmark/bloom_filter.rb 
```

```
benchmarking for 150000 keys with 1.0%, 0.1%, 0.01% error rates
                                               user     system      total        real
BloomFilter m=1437759, k=07 add            0.940000   0.000000   0.940000 (  0.948075)
BloomFilter m=1437759, k=07 include?       0.830000   0.010000   0.840000 (  0.834414)
BloomFilter m=2156639, k=10 add            1.220000   0.000000   1.220000 (  1.227294)
BloomFilter m=2156639, k=10 include?       1.050000   0.010000   1.060000 (  1.052358)
BloomFilter m=2875518, k=13 add            1.500000   0.010000   1.510000 (  1.516086)
BloomFilter m=2875518, k=13 include?       1.260000   0.010000   1.270000 (  1.258877)

BloomFilter m=1437759, k=07 add            158215 ops/s
BloomFilter m=1437759, k=07 include?       179767 ops/s
BloomFilter m=2156639, k=10 add            122220 ops/s
BloomFilter m=2156639, k=10 include?       142537 ops/s
BloomFilter m=2875518, k=13 add             98939 ops/s
BloomFilter m=2875518, k=13 include?       119154 ops/s
```

### ContinuousBloomfilter

``` sh
ruby benchmark/continuous_bloom_filter.rb 
```

```
benchmarking WITHOUT expiration for 150000 keys with 1.0%, 0.1%, 0.01% error rates
                                                            user     system      total        real
ContinuousBloomFilter m=1437759, k=07 add               1.720000   0.000000   1.720000 (  1.733903)
ContinuousBloomFilter m=1437759, k=07 include?          1.630000   0.010000   1.640000 (  1.630668)
ContinuousBloomFilter m=2156639, k=10 add               2.130000   0.010000   2.140000 (  2.142091)
ContinuousBloomFilter m=2156639, k=10 include?          2.160000   0.000000   2.160000 (  2.159395)
ContinuousBloomFilter m=2875518, k=13 add               2.650000   0.010000   2.660000 (  2.655585)
ContinuousBloomFilter m=2875518, k=13 include?          2.570000   0.010000   2.580000 (  2.586032)

ContinuousBloomFilter m=1437759, k=07 add               86510 ops/s
ContinuousBloomFilter m=1437759, k=07 include?          91987 ops/s
ContinuousBloomFilter m=2156639, k=10 add               70025 ops/s
ContinuousBloomFilter m=2156639, k=10 include?          69464 ops/s
ContinuousBloomFilter m=2875518, k=13 add               56485 ops/s
ContinuousBloomFilter m=2875518, k=13 include?          58004 ops/s

benchmarking WITH expiration for 500000 keys with 1.0%, 0.1%, 0.01% error rates
                                                            user     system      total        real
ContinuousBloomFilter m=1437759, k=07 add+include      11.110000   0.040000  11.150000 ( 11.146869)
ContinuousBloomFilter m=2156639, k=10 add+include      14.220000   0.040000  14.260000 ( 14.269583)
ContinuousBloomFilter m=2875518, k=13 add+include      17.600000   0.060000  17.660000 ( 17.665917)

ContinuousBloomFilter m=1437759, k=07 add+include      89711 ops/s
ContinuousBloomFilter m=2156639, k=10 add+include      70079 ops/s
ContinuousBloomFilter m=2875518, k=13 add+include      56606 ops/s
```

## JRuby < 1.7
This has only been tested in Ruby **1.9** mode. JRuby 1.9 mode has to be enabled to run tests and benchmarks. Note that this is not necessary  with JRuby >= 1.7 which is in 1.9 mode by default.

- to run specs use

``` sh
jruby --1.9 -S rake spec
```
- to run benchmarks use 

``` sh
jruby --1.9 benchmark/some_benchmark.rb
```

<a id="references" />
## References ##
- [Bloom filter on wikipedia](http://en.wikipedia.org/wiki/Bloom_filter)
- [Scalable Datasets: Bloom Filters in Ruby](http://www.igvita.com/2008/12/27/scalable-datasets-bloom-filters-in-ruby/)
- [Flow Analysis & Time-based Bloom Filters](http://www.igvita.com/2010/01/06/flow-analysis-time-based-bloom-filters/)
- [Stable Bloom filters](http://webdocs.cs.ualberta.ca/~drafiei/papers/DupDet06Sigmod.pdf)
- [The maths to compute optimal m and k ](http://www.siaris.net/index.cgi/Programming/LanguageBits/Ruby/BloomFilter.rdoc)
- [Producing n hash functions by hashing only once](http://willwhim.wordpress.com/2011/09/03/producing-n-hash-functions-by-hashing-only-once/)
- [Less Hashing, Same Performance: Building a Better Bloom Filter](http://citeseer.ist.psu.edu/viewdoc/download?doi=10.1.1.152.579&rep=rep1&type=pdf)

## Credits
- [Ilya Grigorik](http://www.igvita.com/) for his inspiration with the [Time-based Bloom filters](http://www.igvita.com/2010/01/06/flow-analysis-time-based-bloom-filters/).
- Authors of the [Stable Bloom filters research paper](http://webdocs.cs.ualberta.ca/~drafiei/papers/DupDet06Sigmod.pdf) which also provided inspiration.
- [Robey Pointer](https://github.com/robey) for his [Ruby FNV C extension implementation](https://github.com/robey/rbfnv).
- [Peter Cooper](http://www.petercooper.co.uk/) for inspiration with [his BitField class](http://dzone.com/snippets/bitfield-fastish-pure-ruby-bit).

## Author
Colin Surprenant, [@colinsurprenant](http://twitter.com/colinsurprenant), [http://github.com/colinsurprenant](http://github.com/colinsurprenant), colin.surprenant@gmail.com

## License
Bloombroom is distributed under the Apache License, Version 2.0. 

