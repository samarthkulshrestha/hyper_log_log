# hyper_log_log

a probabilistic cardinality estimation algorithm

### introduction

hyperloglog is an algorithm for the count-distinct problem, approximating the
number of distinct elements in a multiset. calculating the exact
cardinality of the distinct elements of a multiset requires an amount of
memory proportional to the cardinality, which is impractical for very large
data sets. probabilistic cardinality estimators, such as the hyperloglog
algorithm, use significantly less memory than this, but can only approximate
the cardinality. the hyperloglog algorithm is able to estimate cardinalities
of > 109 with a typical accuracy (standard error) of 2%, using 1.5 kb of memory.
hyperloglog is an extension of the earlier loglog algorithm, itself
deriving from the 1984 flajolet–martin algorithm.

### implementation

the algorithm is implemented in C as well as Python. the code for these
implementations in in the `c` and `py` directories respectively. the code is
documented using comments. the following section describes how registers are
represented when hyperloglog is initialised.

#### dense representation

```
========================== Dense representation ==========================

Since register values will never exceed 64 we store them using only 6 bits.
This encoding is diagrammed below:

         b0        b1        b2        b3
         /         /         /         /
    +-------------------+---------+---------+
    |0000 0011|1111 0011|0110 1110|1111 1011|
    +-------------------+---------+---------+
     |_____||_____| |_____||_____| |_____|
        |      |       |      |       |
      offset   m1      m2     m3     m4

     b = bytes, m = registers

The first six bits in b0 are an unused offset. With the exception of byte
aligned registers (e.g. m4), registers will have bits in consecutive bytes.
For example, the register m2 has bits in b1 and b2. The higher order bits
of m2 are in b1 and the lower order bytes of m2 are in the b2.

Getting a register
------------------

Suppose we want to get register m2 (e.g. m=2). First we determine the
indices of the enclosing bytes:

    left byte  = (6*m + 6)/8 - 1                                         (1)
               = 1

    right byte = left byte + 1                                           (2)
               = 2

Next we compute the number of bits of m2 in each byte. The number of right
bits is:

    rb = right bits                                                      (3)
       = (6*m + 6) % 8
       = 2

    lb = left bits                                                       (4)
       = 6 - rb
       = 4

This result is diagrammed below:

        b1         b2
         \         /
    +---------+---------+
    |1111 0011|0110 1110|
    +---------+---------+
          ^^^^ ^^
          /      \
      left bits  right bits

      m2 = "001101"

Move the left bits into the higher order positions:

    +---------+
    |1111 0011|   <-- b1
    |1100 1100|   <-- b1 << rb
    +---------+

Move the right bits to the lower order position:

    +---------+
    |0110 1110|   <-- b2
    |0000 0001|   <-- b2 >> (8 - rb)
    +---------+

Bitwise OR the two bytes, b1 | b2:

    +---------+
    |1100 1100|  <-- b1
    |0000 0001|  <-- b2
    |1100 1101|  <-- b1 | b2
    +---------+

Finally use a mask to remove the bits not part of m2:

    +---------+
    |1100 1101|  <-- b1 | b2
    |0011 1111|  <-- mask to isolate the register bits
    |0000 1101|  <-- m2 = b1 & mask
    +---------+

Setting a register
------------------

Setting a register is similar to getting a register. We determine the
enclosing bytes using (1) and (2). Then the bits of each byte is
computed using (3) and (4). Continuing the previous example using register
m2, at this point we should have:

        b1         b2
         \         /
    +---------+---------+
    |1111 0011|0110 1110|
    +---------+---------+
          ^^^^ ^^
          /      \
      left bits  right bits

      lb = 4, rb = 2
      m2 = "001101"

Let N be the value we want to set. Suppose we want to set m2 to 7 (N=7). We
start by zeroing out the left bits of m in b1 and the rights bits of m in
b2:

    +---------+
    |1111 0011|  <- b1
    |0011 1100|  <- b1 = b1 >> lb
    |1111 0000|  <- b1 = b1 << lb
    |0110 1110|  <- b2
    |1011 1000|  <- b2 = b2 << rb
    |0010 1110|  <- b2 = b2 >> rb
    +---------+

Now that we have made space for m2, we need to set the new bits. We can get
new bits by simplying shifting N:

     new right bits
           \
           vv
   +---------+
   |0000 0111|  <- N=7
   +---------+
      ^^ ^^
       \ /
   new left bits

   nlb = new left bits
       = N >> rb
       = 7 >> 2

   nrb = new right bits
       = N << (8 - rb)
       = 7 << 6

We can now set the left byte b1 using bitwise OR:

   +---------+
   |1111 0000|  <- b1
   |0000 0001|  <- nlb
   |1111 0001|  <- b1 | nlb
   +---------+

Setting the right byte b2 using bitwise OR:

   +---------+
   |0010 1110|  <- b2
   |1100 0000|  <- nrb
   |1110 1110|  <- b2 | nrb
   +---------+

The bytes have been updated so we're done. The final result is shown
below:

        b1         b2
         \         /
    +---------+---------+
    |1111 0001|1110 1110|
    +---------+---------+
          ^^^^ ^^
          /      \
      left bits  right bits

      lb = 4, rb = 2
      m2 = "000111"

```

#### sparse representation

```
========================== Sparse representation ==========================

When a HyperLogLog is created its register values are initialized to zero.
Because the registers share the same value it is inefficient to store
them individually. Instead only non-zero registers are stored. These
registers are stored as nodes in a sorted link list. For example the
registers

    +-+-+-+-+-+-+-+-+
    |0|3|0|0|1|1|0|2|
    +-+-+-+-+-+-+-+-+

are represented with the following linked list (sorted by index):

           index
             |
             v
    +---+   +---+   +---+   +---+
    |1,3|-->|4,1|-->|5,1|-->|7,2|
    +---+   +---+   +---+   +---+
               ^
               |
             value

To avoid the worst case of traversing the linked list every time add()
is called a temporary register buffer is used to store the new register
values. When the buffer is full it is sorted and the linked list is
updated. Because both the list and buffer are sorted this update can be
done in one pass.

Eventually the linked list grows too large to save memory. When this
happens the HyperLogLog switches to a dense representation.
```

### contribute

 [![pull requests welcome](https://img.shields.io/badge/prs-welcome-brightgreen.svg?style=flat-square)](https://makeapullrequest.com)
 [![c style guide](https://img.shields.io/badge/c-style%20guide-blue?style=flat-square)](https://cs50.readthedocs.io/style/c/)

+ i <3 pull requests and bug reports!
+ don't hesitate to [tell me my code-fu sucks](https://github.com/samarthkulshrestha/hyper_log_log/issues/new), but please tell me why.

## license

hyper_log_log is licensed under the mit license.

copyright (c) 2023 samarth kulshrestha.
