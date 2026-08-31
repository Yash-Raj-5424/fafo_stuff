- cloudflare owns 1.1.1.1 and its DNS cache stores over 250B cache entries.

- they optimized how cache entries are stored in memory and the results: per-entry footprint reduced by 50%, 100TB memory freed, insert throughput rose by 43%, lookup latency dropped by 19%

- Pineapple starts with empty cache and fills to max as DNS queries arrive then evict older or less popular ones

- the key of cache entry identifies what was queried and value is the DNS response. the response includes answer, authority, metadata etc.

- they tracked memory - recorded no & size of allocations per cache entry also the monitored throughput, lookup latency across the full cache flow.

- tracking just these wasn't all as process memory depends on other factors like traffic mix, cache occupancy, allocator state etc. so they measured resident memory across prodxn instances during the rollout

**cost of capacity**
- Vec<T> - ptr to heap-allocated data, curlength and total capacity. it appends if there's space otherwise reallocates as needed. the capacity field isn't of much use and wasted 8 bytes per Vec.

- so they used Box<[T]> which didn't needed capacity field. each cache entry was storing 8 Vec and String fields so it saved 8x8 = 64 bytes per entry

- for response they kept single list with offsets for start of each section (used U16 that takes 2 bytes instead of 8-byte ptr for each offset)

- also packed several boolean fields into single bitflag

- for repeated owner field they used 2-byte ptr to the 1st occurrence. so no twice encoding

- the enums in rust is as large as its largest variant. they boxed the larger variants, moving them to separate heap allocation and the enums store an 8-byte ptr to the heap where data takes actual size it requires

- boxing costed allocator overhead and poor memory locality (earlier -> values in single continuous allocation). the boxed variants live in separate heap region and reading them bring CPU overhead (following a ptr to heap for each boxed one).

- storing the full DNS response in **wire format** could have drawbacks of parsing the full msg body on every lookup.

- they stored just the record data as raw bytes keeping rest of cache entry as structured fields. instead of list of parsed ones, they used Box<[u8]> containing each record encoded as 2-byte length prefix followed by its raw bytes.

- this improved CPU cache locality and now data was packed contiguously.

- most records could be copied from buffer while building DNS response from cached records. earlier each parsed record had to be serialized field by field back to DNS wire format. now the encoded bytes are copied directly(for A, AAA, TXT, DNSSEC record types - these were majority).

- they write into a reusable buffer to build the record data buffer that persists across cache insertions.

- once the records are in the buffer, they allocate a Box<[u8]> and memcpy the data into it. this replaced separate allocation for each boxed record with one allocation. this increased the cache insert throughput by 13%


*blog link*
>https://blog.cloudflare.com/dns-cache-memory-optimization-1111/
