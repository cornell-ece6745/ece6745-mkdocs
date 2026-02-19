
ECE 6745 Project 2 Ideas
==========================================================================

For project 2, you will be designing, implementing, testing, evaluating,
and fabricating an accelerator which connects to a TinyRV2 processor. The
processor can read and write _accelerator registers_ using CSR
instructions to send data to and receive data from the accelerator. Here
is the top-level system diagram.

![](img/project2-proc-xcel.png)

Even though you will only be taping out the accelerator, we still want to
do a rigorous comparative analysis of the system as a whole. Here are
some project ideas.

 - **Compression/Decompression Accelerator:** Create an accelerator that
   implements some kind of advanced compression algorithm. The processor
   would send over 32-64B of data by writing 8-16 accelerator registers
   and then read back the result by reading a different set of 8-16
   accelerator registers. The accelerator could be for compression,
   decompression, or both.

 - **Cryptographic Hashing Accelerator:** Create an accelerator for for a
   cryptographic hash function like SHA-1 or SHA-2. The processor would
   send over the 32-64B of data by writing 8-16 accelerator registers and
   then read back the hashed version by reading a different set of 8-16
   accelerator registers.

 - **Encryption/Decryption Accelerator:** Create an accelerator that
   implements some kind of encryption or decryption algorithm such as
   AES. The processor would send over 32-64B of data by writing 8-16
   accelerator registers and then read back the result by reading a
   different set of 8-16 accelerator registers. The accelerator could be
   for encryption, decryption, or both.

 - **Floating-Point Adder or Multiplier:** Create a hardware floating
   point unit (FPU). The procecssor would write the operands to two
   accelerator registers and read the result from a third accelerator
   register. The FPU could potentially support both addition and
   multiplication; the processor might write a fourth accelerator
   register to indicate which operation to perform.

 - **DNA Sequence Alignment Accelerator:** Create an accelerator for DNA
   sequence alignment using a dynamic programming algorithm such as
   Smith-Waterman. The processor would send over two DNA sequences packed
   into 32-bit words (two bits per basepair, so 16 basepairs per 32-bit
   word, so up a 256-basepair sequence can be written using 16
   accelerator registers). The processor would read back the alignment
   score and potentially also read back the actual alignment. Such an
   accelerator could eventually be used to process a tile of a much
   larger sequence alignment.

 - **Image Processing Accelerator:** Create an accelerator for corner or
   feature detection or even optical flow might be interesting. The
   processor would send over a small image. For example, assuming 4-bit
   gray-scale pixels, the processor could send over a 16x16 image block
   by writing 32 accelerator registers (i.e., a 16x16 image block has 256
   pixels, eight pixels packed into each 32-bit word). The processor can
   read back the output image by reading these same 32 accelerator
   registers. Such an accelerator could eventually be used to process a
   tile of a much larger image.

 - **Dense Matrix Multiplication Accelerator:** Create a systolic array
   to accelerate integer matrix multiplication. The processor would send
   over two small matrices. For example, assuming 8-bit elements pixels,
   the processor could send over an 8x8 matrix by writing 16 accelerator
   registers (i.e., a 8x8 matrix has 64 elements, four elements packed
   into each 32-bit word). The processor would need to write two matrices
   and then read the result by reading some of these same registers.

 - **Sparse Matrix Vector Accelerator:** Create a accelerator for sparse
   matrxi-vector multiplication. Assume the metrix is stored in CSR
   format and the vector is dense. The accelerator would need to operate
   on very small matrices; just what is feasible to write through
   accelerator registers.

- **FFT Butterfly Accelerator:** Create an accelerator to accelerate the
   butterfly operation in an FFT. The processor would need to send over
   the data but also potentially the twiddle factors. Ideally the FFT
   butterfly accelerator would be evaluated in the context of a
   performing an FFT on a small input array.

 - **CORDIC Accelerator:** Create an accelerator for trigonometric
   functions by using the CORDIC algorithm. The processor might send over
   a vector and an angle by writing several four accelerator registers
   (i.e., three to specify the vector, one to specify the angle) and then
   read the result (i.e., the rotated vector) by reading three additional
   accelerator registers.

 - **Integer Square Root Accelerator:** Create an accelerator that can
   find the square root of an integer value using an iterative approach.

