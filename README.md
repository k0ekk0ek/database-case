[1]: https://blog.nlnetlabs.nl/adapting-radix-trees

# The case for a new database

Research shows database performance can be much improved. A blog post has been
published on the subject: [Adapting Radix Trees][1]. The prototype proves
performance for database lookups can be much improved using vector
instructions found in modern processors while even using slightly less memory
than the red-black tree that is currently in use. After developing simdzone
and gaining more insight, I'm confident peformance can by improved upon even
further. (e.g. by vectorizing the key transformation algorithm).

While the database refresh will improve performance and bring down memory
use some, I believe we can massively improve in both areas by modifying the
contents of the database itself. The "trick", in my opinion, is to simplify
the internal representation of the data to more closely resemble wire-format
The in-memory representation of DNS RRs splits up RDATA and uses a generic
loop to marshall data into DNS wire format. A simplified version in pseudocode
is shown below.

```
// associate each field type with an identifier. RDATA_WF_COMPRESSED_DNAME
// is 0 (zero), RDATA_WF_UNCOMPRESSED_DNAME is 1 (one), etc.
enum rdata_wireformat {
  RDATA_WF_COMPRESSED_DNAME, /* Possibly compressed domain name. */
  RDATA_WF_UNCOMPRESSED_DNAME, /* Uncompressed domain name. */
  RDATA_WF_LITERAL_DNAME, /* Literal (not downcased) dname. */
  RDATA_WF_BYTE, /* 8-bit integer. */
  RDATA_WF_SHORT, /* 16-bit integer */
  RDATA_WF_LONG, /* 32-bit integer */
  /* ... */
};

typedef enum rdata_wireformat rdata_wireformat_type;

// define a basic structure to hold all information associated with an RRTYPE.
struct rrtype_descriptor {
  uint16_t type; /* RR type */
  const char *name; /* Textual name. */
  uint32_t minimum; /* Minimum number of RDATAs. */
  uint32_t maximum; /* Maximum number of RDATAs. */
  // a sequence of RDATA fields, one wireformat identifier per field.
  uint8_t wireformat[64]; /* rdata_wireformat_type */
  /* ... */
};

typedef struct rrtype_descriptor rrtype_descriptor_type;

// define one descriptor per RRTYPE indexed by type code.
rrtype_descriptor_type rrtype_descriptors[259] = {
  /* ... */
  { TYPE_A, "A", 1, 1, { RDATA_WF_A }, /* ... */ },
  /* ... */
  { TYPE_SOA, "SOA, 7, 7,
    { RDATA_WF_COMPRESSED_DNAME, RDATA_WF_COMPRESSED_DNAME, RDATA_WF_LONG,
      RDATA_WF_LONG, RDATA_WF_LONG, RDATA_WF_LONG, RDATA_WF_LONG },
    /* ... */
  }
  /* ... */
};

// when a query is answered, for each RR
foreach(rrset in rrsets) {
  foreach(rr in rrset) {
    foreach(rdata_atom in rdata) {
      if (rdata_atom->type == COMPRESSED_NAME) {
        // dereference pointer to get domain object from table
        // write name using compression
      } else if (rdata_atom->type == UNCOMPRESSED_NAME) {
        // dereference pointer to get domain object from table
        // write name as-is
      } else {
        copy_rdata_to_packet(packet, rdata_item);
      }
    }
  }
}
```

The reason the RDATA is compiled of so-called rdata\_atom(s), is so that
records for which a domain name in RDATA is to be looked up, the code only
needs to dereference a pointer rather then do a complete lookup, which in
itself is quite an improvement. Pointers to each rdata atom are stored in
a vector of \<n\> elements that is allocated separately from the RR object
that contains the owner, type, class and ttl.

A diagram of how a SOA RR is stored in memory:

```

    0  1  2  3  4  5  6  7  0  1  2  3  4  5  6  7
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
  | owner (pointer to domain object)              | -> <domain>
  |                                               |
  |                                               |
  |                                               |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
  | rdatas (pointer to sequence of pointers)      | -\
  |                                               |  |
  |                                               |  |
  |                                               |  |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+  |
  | ttl (uint16_t)                                |  |
  |                                               |  |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+  |
  | klass (uint16_t)                              |  |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+  |
  | type (uint16_t)                               |  |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+  |
  | rdata_count (uint16_t, number of rdatas)      |  |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+  |
                                                     |
                          /--------------------------/
                          |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
  | MNAME* (struct domain *)                      | -> <domain>
  |                                               |
  |                                               |
  |                                               |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
  | RNAME* (struct domain *)                      | -> <domain>
  |                                               |
  |                                               |
  |                                               |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
  | SERIAL* (void*)                               | ----------\
  |                                               |           |
  |                                               |           |
  |                                               |           |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+           |
  | REFRESH* (void*)                              | --------\ |
  |                                               |         | |
  |                                               |         | |
  |                                               |         | |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+         | |
  | RETRY* (void*)                                | ------\ | |
  |                                               |       | | |
  |                                               |       | | |
  |                                               |       | | |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+       | | |
  | EXPIRE* (void*)                               | ----\ | | |
  |                                               |     | | | |
  |                                               |     | | | |
  |                                               |     | | | |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+     | | | |
  | MINIMUM* (void*)                              | --\ | | | |
  |                                               |   | | | | |
  |                                               |   | | | | |
  |                                               |   | | | | |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+   | | | | |
                                                      | | | | |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+   | | | | |
  | MINIMUM (uint32_t, 64-bit aligned)            | <-/ | | | |
  |                                               |     | | | |
  |                                               |     | | | |
  |                                               |     | | | |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+     | | | |
                                                        | | | |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+     | | | |
  | EXPIRE (uint32_t, 64-bit aligned)             | <---/ | | |
  |                                               |       | | |
  |                                               |       | | |
  |                                               |       | | |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+       | | |
                                                          | | |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+       | | |
  | RETRY (uint32_t, 64-bit aligned)              | <-----/ | |
  |                                               |         | |
  |                                               |         | |
  |                                               |         | |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+         | |
                                                            | |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+         | |
  | REFRESH (uint32_t, 64-bit aligned)            | <-------/ |
  |                                               |           |
  |                                               |           |
  |                                               |           |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+           |
                                                              |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+           |
  | SERIAL (uint32_t, 64-bit aligned)             | <---------/
  |                                               |
  |                                               |
  |                                               |
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

The MNAME and RNAME fields will always be pointers as it is the most effective
method to quickly dereference the data. Apart from that, referencing an object
in this way allows for deduplication as the name is stored only once. Note
that there is quite a bit of overhead involved in storing a 32-bit integers.
More precisely, if MNAME and RNAME are stored as pointers (64-bit on a 64-bit
architecture), the data stored for a SOA RR is 5 * 32-bits + 2 * 64-bits, or
20 bytes. The overhead involved with the current strategy is significant. As
each field is allocated separately and always aligned up to 64-bits,
5 * 32-bits of space is wasted just to store the actual data. As each field is
individually dereferenced, another 64-bits per field is wasted. If we include
the pointer to the vector, another 64-bits, the total amount of space wasted
is: 5 * 16-bits (size in atom) + 5 * 32-bits + 7 * 64-bits + 1 * 64-bits, or
94 bytes. While the overhead is a little lower for NS RRs (1 * 64-bits +
1 * 64-bits, or 16 bytes), the total number is quite significant for e.g. .com
(~160 million domains). If each domain had just one NS record, the space saved
would be ~2.4GB.

The choice for a more-or-less generic descriptor table was made to make it
easy to add support for new RRs. During development of simdzone, I initially
focused on conversion between formats using a somewhat generic descriptor
table too, but identified another, in my opinion better, way to handle
conversion. I still use a table, but instead of converting fields in a serial
manner by calling the corresponding function, I implement the functionality
directly in the function. This is possible because the function knows exactly
how the data is structured. The obvious downside is lots and lots of code
duplication. At least, that’s what you expect, right? Not so much. C99
introduced the inline keyword (we use it alread), which allows the programmer
to write a function, but instead of actually calling it as a function the
compiler places the function code directly in the code. The benefit here being
that we avoid the overhead of a function call. As the function knows exactly
what fields are located in which offsets, we can greatly simplify the
individual code paths without all that much extra code. A lot of RR types do
not even require us to individually address fields.

A simplified example without boundary checks etc:

```c
struct rr {
  struct domain *owner;
  uint16_t type;
  uint16_t class;
  uint32_t ttl;
  uint16_t rdlength;
  // rdata
};

__attribute__((always_inline))
static inline const uint8_t *rr_rdata(const struct rr *rr)
{
  return ((uint8_t *)rr) + sizeof(rr);
}

static int32_t write_a_rr(const struct rr *rr, packet_t *packet)
{
  copy_rdata(packet->octets, rr_rdata(rr), rr->rdlength);
  return 0;
}

static int32_t write_soa_rr(const struct rr *rr, packet_t *packet)
{
  struct domain *mname, *rname;
  const uint8_t *rdata = rr_rdata(rr);
  memcpy(&mname, rdata, sizeof(mname));
  memcpy(&rname, rdata+sizeof(mname), sizeof(rname));
  copy_compressed_name(packet, mname);
  copy_compressed_name(packet, rname);
  copy_rdata(packet->octets,
             rdata + (sizeof(mname) + sizeof(rname)),
             rr->rdlength - (sizeof(mname) + sizeof(rname)));
  return 0;
}

typedef int32_t(*write_rr_rdata_t)(const struct rr *rr, packet_t *packet);

static const write_rr_rdata_t writers[258] = {
  0,
  write_a_rr,
  0,
  0,
  0,
  0,
  write_soa_rr,
  // ...
};

static int32_t write_rr_rdata(const struct rr *rr, packet_t *packet)
{
  if (rr->type < 258)
    return writers[rr->type](rr, packet);
  // do some special handling for unknown type or handle error, etc
  return -1;
}
```

The direct benefit with a simpler code path, is a reduction in data
dependencies and branches. As previous research has shown, avoiding branching
on unpredictable data (read individual rdata fields), increases throughput.
The removal of the for loop removes data dependencies and the overhead of
calling one function per field.

If I run the simplified [benchmark](benchmark.c) (compiled with `-O3`) I get
the following results:

```
generated rr size: 66088704
generated fast rr size: 24039424
rdtsc_overhead set to 12
rr                            	:     6.000 cycle/op (best)   28.407 cycle/op (avg)
written: 23046816
fast_rr                       	:     4.000 cycle/op (best)   24.770 cycle/op (avg)
written: 23046816
```

Gut feeling: although the `fast_rr` method is consistently faster by a small
margin, I think it will be significantly faster if we benchmark code that more
closely resembles actual NSD behavior. Even if it's only marginally faster,
the amount of memory saved per-RR is (IMHO) worth the effort.
