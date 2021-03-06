#AddressSpacesUsingBuffers
*Proposal For OpenCL address space support using java Buffers instead of arrays. Updated Dec 8, 2011 by frost.g...@gmail.com*
The general idea is to have a AS_PRIMTYPE_Buffer for each AS=address space and PRIM=primitive type. Here is an example for LocalFloatBuffer which would be a buffer for floats that got mapped to OpenCL local address space.

As with normal FloatBuffers, the float elements are accessed using get and put methods

Although a LocalFloatBuffer conceptually exists only for the lifetime of a workgroup, it is still constructed in the enclosing Kernel, not in the Kernel.Entry.run method. (Aparapi does not support constructing new objects inside the Kernel.Entry.run method).

A typical declaration would be:

    LocalFloatBuffer locbuf = new LocalFloatBuffer{12);
The argument 12 here means that 12 floats would be used by each workitem in the workgroup. So the total buffer would be LocalSize*12 floats. Aparapi would at runtime allocate a total local OpenCL buffer to be this size. Note how this removes the need for the programmer to specify localSize anywhere.

Note: For each Kernel.Entry.execute(globalSize) call, the runtime will determine an appropriate workgroup size, also called localSize, depending on the capabilities of the device, and on the globalSize. The localSize will always evenly divide the globalSize, in other words all workgroups for an execute context will be the same size. A workitem can determine localSize by calling getLocalSize().

Because workitems operate simultaneously and in an undetermined order, workitems will generally only use put on its own portion of the LocalFloatBuffer between the LocalBarriers, and will generally only use get outside the LocalBarriers.

Some example code (from NBody) follows. Here each workitem copies a "BODY" consisting of 4 floats. The global array contains 4*globalSize floats, and we want to iterate thru this global array, copying it into local memory and operating on it there. This will take globalSize/localSize "tiles". For each tile, each workitem fills in one "BODY"'s worth or 4 elements

      // outside run method...
      final int BODYSIZE = 4;
      LocalFloatBuffer pos_xyzm_local = new LocalFloatBuffer(BODYSIZE);
      //
      // inside run method...
      int numTiles = globalSize / localSize;
      for (int i = 0; i < numTiles; ++i) {
         // load one tile into local memory
         int idx = i * localSize + localId;  // index into a global memory array
         localBarrier();
         pos_xyzm_local.put(localId * BODYSIZE + 0, pos_xyzm[idx * BODYSIZE + 0]);
         pos_xyzm_local.put(localId * BODYSIZE + 1, pos_xyzm[idx * BODYSIZE + 1]);
         pos_xyzm_local.put(localId * BODYSIZE + 2, pos_xyzm[idx * BODYSIZE + 2]);
         pos_xyzm_local.put(localId * BODYSIZE + 3, pos_xyzm[idx * BODYSIZE + 3]);
         // Synchronize to make sure data is available for processing
         localBarrier();

         // now the entire LocalFloatBuffer has been filled.
         // each workitem might use the entire Buffer
         // which consists of localSize BODYs
         for (int j = 0; j < localSize; ++j) {
            float r_x = pos_xyzm_local.get(j * BODYSIZE + 0) - myPos_x;
            float r_y = pos_xyzm_local.get(j * BODYSIZE + 1) - myPos_y;
            float r_z = pos_xyzm_local.get(j * BODYSIZE + 2) - myPos_z;
            // ...etc