# ESIMD Examples

This folder contains simple ESIMD examples. The main purpose of having them
is to show the basic ESIMD APIs in well known examples.

1) The most basic example - ["sum_two_arrays"](./sum_two_arrays.md).
   
   Please see the full source here: ["sum_two_arrays"](./sum_two_arrays.md).
   ```c++
   float *a = malloc_shared<float>(Size, q); // USM memory for A
   float *b = new float[Size];               // B uses HOST memory
   buffer<float, 1> buf_b(b, Size);

   // Initialize 'a' and 'b' here.
    
   // Compute: a[i] += b[i];
   q.submit([&](handler &cgh) {
     auto acc_b = buf_b.get_access<access::mode::read>(cgh);
     cgh.parallel_for(Size / VL, [=](id<1> i) [[intel::sycl_explicit_simd]] {
       auto element_offset = i * VL;
       simd<float, VL> vec_a(a + element_offset); // Pointer arithmetic uses element offset
       simd<float, VL> vec_b(acc_b, element_offset * sizeof(float)); // accessor API uses byte-offset

       vec_a += vec_b;
       vec_a.copy_to(a + element_offset);
     });
   }).wait_and_throw();
   ```
2) Calling ESIMD from SYCL using invoke_simd - ["invoke_simd"](./invoke_simd.md).
   
   Please see the full source code here: ["invoke_simd"](./invoke_simd.md)
   ```c++
   [[intel::device_indirectly_callable]] simd<int, VL> __regcall scale(
     simd<int, VL> x, int n) SYCL_ESIMD_FUNCTION {
     esimd::simd<int, VL> vec = x;
     esimd::simd<int, VL> result = vec * n;
    return result;
   }

   int main(void) { 
     int *in = new int[SIZE];
     int *out = new int[SIZE];
     buffer<int, 1> bufin(in, range<1>(SIZE));
     buffer<int, 1> bufout(out, range<1>(SIZE));

     // scale factor
     int n = 2;

     sycl::range<1> GlobalRange{SIZE};
     sycl::range<1> LocalRange{VL};
    
     q.submit([&](handler &cgh) {
      auto accin = bufin.get_access<access::mode::read>(cgh);
      auto accout = bufout.get_access<access::mode::write>(cgh);

      cgh.parallel_for<class Scale>(
          nd_range<1>(GlobalRange, LocalRange), [=](nd_item<1> item) {
            sycl::sub_group sg = item.get_sub_group();
            unsigned int offset = item.get_global_linear_id();

            int in_val = sg.load(accin.get_pointer() + offset);

            int out_val = invoke_simd(sg, scale, in_val, uniform{n});

            sg.store(accout.get_pointer() + offset, out_val);
          });
    });
    ```
3)  Calling ESIMD from SYCL using invoke_simd and Shared Local Memory (SLM) - ["invoke_simd_slm"](./invoke_simd_slm.md).
  
    Please see the full source code here: ["invoke_simd_slm"](./invoke_simd_slm.md)
    ```c++
     [[intel::device_indirectly_callable]] SYCL_EXTERNAL void __regcall invoke_slm_load_store(
     local_accessor<int, 1> *local_acc, uint32_t slm_byte_offset, int *in, int *out,
     simd<uint32_t, VL> global_byte_offsets) SYCL_ESIMD_FUNCTION {
      esimd::simd<uint32_t, VL> esimd_global_byte_offsets = global_byte_offsets;
      // Read SLM in ESIMD context.
      auto local1 = esimd::block_load<int, VL>(*local_acc, slm_byte_offset);
      auto local2 = esimd::block_load<int, VL>(*local_acc, slm_byte_offset + 
                                               LOCAL_RANGE * sizeof(int));
      auto global = esimd::gather(in, esimd_global_byte_offsets);
      auto res = global + local1 + local2;
      esimd::scatter(out, esimd_global_byte_offsets, res);
     }

     int main(void) {
       auto *in = malloc_shared<int>(GLOBAL_RANGE, q);
       auto *out = malloc_shared<int>(GLOBAL_RANGE, q);
       ...
       q.submit([&](handler &cgh) {
         auto local_acc = local_accessor<int, 1>(LOCAL_RANGE * 2, cgh);
         cgh.parallel_for(nd_range, [=](nd_item<1> item) {
           uint32_t global_id = item.get_global_id(0);
           uint32_t local_id = item.get_local_id(0);
           // Write/initialize SLM in SYCL context.
           auto local_acc_copy = local_acc;
           local_acc_copy[local_id] = global_id * 2;
           local_acc_copy[local_id + LOCAL_RANGE] = global_id * 3;
           item.barrier();

           uint32_t la_byte_offset = (local_id / VL) * VL * sizeof(int);
           uint32_t global_byte_offset = global_id * sizeof(int);
           sycl::sub_group sg = item.get_sub_group();
           // Pass the local-accessor to initialized SLM memory to ESIMD context.
           // Pointer to a local copy of the local accessor is passed instead of a local-accessor value now
           // to work-around a known/temporary issue in GPU driver.
           auto local_acc_arg = uniform{&local_acc_copy};
           invoke_simd(sg, invoke_slm_load_store, local_acc_arg,
                       uniform{la_byte_offset}, uniform{in}, uniform{out},
                       global_byte_offset);
         });
       })
    ```

6) TODO: Add more examples here.
