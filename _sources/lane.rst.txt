Lane
=====

Ara Vector Register File
-------------------------

The Vector Register File (VRF) of Ara is implemented using a set of single ported (1RW) memory banks. The width of each bank is 64-bits which is the same as the width of the datapath of each lane. There are eight banks per lane resulting in eight single ported memory banks per lane. In the RVV ISA there are 32 vector registers. The size of each vector register, in bits, is implementation dependent (minimum = 128 bit).  Let's look at the size of VRF by using an example. 

Size of Vector register  (VLEN) = 4096 bit

Size of Vector Register File = 4096 * 32 = 217 bits = 16 KB

Number of Lanes in Ara can vary from 1 to 16 (implementation dependent). So for a 4 lane system the VRF will be divided into 4 lanes with 4KB of memory per lane.

..  image:: /images/VRFSize.png
    :alt: VRF Size pre Lane
    :scale: 50
    :align: center

|

Benefits of splitting VRF across lanes:

* As lanes are our processing elements so most computation constrained within one lane makes hardware implementation easy.

* Removes the dependency of the execution units  on the number of lanes which makes Ara scalable.

* As VRF is internal to the lane routing between VRF and lane execution logic is simplified.


Division of VRF into Banks
^^^^^^^^^^^^^^^^^^^^^^^^^^^
As indicated earlier the portion of VRF inside a lane is further divided into eight 64-bit wide banks. Each bank is implemented using a single-ported SRAM memory. Being a single ported memory it has only one address port for both reads and writes. In steady state, under worst case conditions, 5 banks are accessed simultaneously in order to support predicated multiply-accumulate instructions (which require 3 source registers plus the mask register resulting in 4 reads, and one destination register -resulting in a single write). 


..  image:: /images/VRFPerLanePerBank.png
    :alt: VRF Size per Banks
    :align: center

|

Continuing with the earlier example, where each of the four lanes has 4KB of memory, each lane will as such have 8 banks of 512B (4096B / 8 =  512B) memory. Given that each bank is 64-bits wide there will be 64 (512B/8B = 64) 8B wide addressable locations within each bank. 

Since each lane has  4KB of VRF memory the address bus, going to each lane,  will be 12-bits wide. Since there are 8 banks within a lane the least significant 3 bits of the address are used for addressing the 8 banks and the remaining 9-bits are used to address the byte within the bank. Of these remaining 9-bits, 6-bits are used to address 1-of-the-64 64-bit locations within the bank and the remaining 3 bits of the address are used to address the individual byte within the 64-bits.

..  figure:: /images/addressPerLane.png
    :alt: Address distribution inside a lane
    :scale: 40
    :align: center

|

Translating Vector Register Number to SRAM memory Bank
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Vector register number fields ranging in value from 0 to 31 (5-bits) are present in the instruction. They are decoded and used to determine the starting address of that vector register in the VRF memory. Within a lane the vector registers are stored as a contiguous set of bytes. The register number (vid) is multiplied by a factor which is equal to the number of bytes in a single vector register divided by the number of lanes. The resulting number gives the starting address of the vector register in the VRF memory. 

For normal cases the starting address of every vector register is a multiple of 8 given that the multiplying factor that is used is a  power of 2 and  greater than or equal to 8. For our continuing example this factor is equal to 128 [(4096/8)/4].  In some cases where VLEN is small e.g.128, and number of lanes are high e.g. 16 this is not true.  So if the address is a multiple of 8  then the least significant three bits of the address are always 0.  As indicated earlier the least significant 3 bits of the address are used to select the bank. As such every vector register has its first element (its starting point) in the first bank i.e., bank 0 and this will cause bank conflicts which are discussed later.

Continuing with our example the following table gives the starting address in the VRF memory for all 32 vector registers. As can be seen the last nibble of every address is 0 implying that the last 3 address bits are always 0 and as such always pointing to bank 0.


.. csv-table:: Starting Address of 32 Vector Register in VRF Memory
   :file: /documents/Vector_Starting_Addr.csv
   :header-rows: 1

This arrangement causes bank conflicts because banks are composed of single ported memories and as such only a single request to read or write an operand can be sent to a bank. However functional units can request multiple operands simultaneously and as such need to access the same bank at the same time causing bank conflicts.

To resolve bank conflicts Ara uses a weighted round robin priority arbiter per bank. Mask register (v0) has the highest priority, followed by reading of operands A, B, C, and then writing a destination register.


Organization of Elements in Vector Register File
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Elements of a  vector register are mapped to consecutive lanes in the VRF. Each lane’s  datapath width is 64 bits which matches the width of an individual register file bank. The  figure below shows 4 lanes, numbered 0-3  with each lane having 8 bytes (64 bits) and the consecutive bytes numbered 0 - 1F. Since there are functional units present in each lane, groups of elements can be processed in parallel. For example if we have four 64-bit elements and four lanes then each element is placed in a different lane shown by the numbers 0-3 below. All 4 of these elements can be accessed simultaneously from the VRF and can be operated upon in parallel.


..  figure:: /images/VLEN512_SEW64_Lane4.png
    :alt: Elements split across lanes
    :align: center

|

Continuing with the previous arrangement across lanes, elements within a single lane are also arranged in an order such that when the element width changes the mapping between some of the elements and lanes remains the same. Due to this, the mapping between consecutive bytes in a lane and consecutive bytes of an element is not the same. This is resolved by using shuffle logic described in the next section.  

As an example with a vector length of 256B (2048-bits) and 4 lanes each of which are  64-bits wide the mapping of bytes for various sizes of SEW are shown below:

..  figure:: /images/elem_lane_organization.png
    :alt: Organiation of elements in Lanes with variation in SEW
    :align: center

|

For the first case (SEW = 64) the arrangement is simple and obvious. Each 64-bit element is mapped into a different lane and the bytes of each element are contiguously stored within the lane. For the second case (SEW = 32) elements 0, 1, 2 and 3 are mapped in the lower half (32-bit) of lane 0 through lane 3. Element 4 then gets mapped to the upper half of lane 0 and so on until element 7 gets mapped to the upper half of lane 3. It is important to note that the least significant byte of  elements 0, 1, 2 and 3 for both cases of SEW are mapped to the same byte location within the corresponding lanes.  This process continues to be repeated for the remaining values of SEW. 

In the case of SEW = 16 and 8 there is an additional thing to observe. When we start from lane 0 to start assigning the first byte of a new element we also observe which half of the lane (upper or lower 32-bits) the last element in the lane was assigned.  For the new assignment we pick the half which was not used the previous time. So in the case of SEW = 16  when it comes time to assigning element 8 it is placed in Lane 0’s byte position 2 (lower half of lane 0) and not byte position 6 (upper half of lane 0)  since the last element assigned in lane 0, element 4, was assigned in byte position 4 (upper half of lane 0).      

The above arrangement of elements and their corresponding bytes in the VRF is constrained within the lane. Data outside the lane i.e in memory, is arranged with bytes packed simply from the least-significant byte to the most-significant byte in increasing memory addresses. As such when data is moved between the memory and the VRF bytes get shuffled/de-shuffled to match the appropriate required ordering. These two ordering of elements and bytes is referred to as Lane Organization of bytes and Natural Packing of bytes. 

**Lane Organization:** The manner in which elements and their corresponding bytes are stored in the VRF as discussed above.

**Natural Packing:** The elements and bytes packed in memory with the least-significant bye to the most-significant byte in increasing memory addresses.


Shuffle Logic
^^^^^^^^^^^^^^
The shuffle/de-shuffle logic sits between the memory subsystem and the VRF as shown below. When data is moved from memory to the VRF (via a load instruction) it gets shuffled from the Natural Packing arrangement to the Lane Organization arrangement. Similarly When data is moved from VRF to memory (via a store instruction) it gets de-shuffled from the  Lane Organization to the Natural Packing arrangement.


..  figure:: /images/shuffle_interconnect.png
    :alt: Shuffle logic interconnect between memory and VRF
    :scale: 40
    :align: center

|

The mapping of bytes from Natural Packing to Lane Organization for 4 lanes and SEW of 16 is shown in the figure below.  For element 0, byte indices are the same, 0 & 1, for Natural Packing and Lane Organization. Element 1 is mapped  to byte index 8 in the VRF with its two bytes in indices 8 & 9 in the VRF. Shuffle logic takes the sequential bytes from memory as shown in the natural Packing row and converts it into the Lane Organization arrangement as shown in the Lane Organization row. De-shuffle logic does the opposite. These mappings are shown with arrows in the diagram for some of the elements.


..  figure:: /images/shuffling_logic_SEW16.png
    :alt: Shuffle logic for SEW=16
    :align: center

|

The arrangement of  the shuffle/de-shuffle logic is a function of SEW. This means that when a vector is moved between Memory and the VRF or vice versa, bytes get shuffled/de-shuffled based on the value of the vector’s SEW (or EEW). As such in addition to the bytes of the vector being stored in the VRF it also gets tagged, in hardware, with its SEW (or EEW.) This tag is subsequently used by the shuffle/de-shuffle logic when data is moved around.   

..  figure:: /images/shuffling_logic_vary_sew.png
    :alt: Shuffling logic with variation of sew
    :align: center

|

**Note:** Next two section are pending for review. Thanks


Mapping of elements to Vector Register File
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Vector Register File and Operand-Deliver Interconnect
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
