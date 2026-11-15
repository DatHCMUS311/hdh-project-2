# Báo cáo Bài 1 & Bài 3 – Lab Page Tables (xv6-riscv)

## 1. Thông tin chung
- **Nhánh thực hiện**: `truong`
- **File thay đổi**: `kernel/vm.c`
- **Các bài đã hoàn thành**:
  - Bài 1: Duyệt và in bảng trang hai cấp (`vmprint`)
  - Bài 3: Sử dụng superpages cho bảng trang kernel (`kmappages` / `kvmmap`)

---

## 2. Bài 1: Duyệt và in bảng trang hai cấp (`vmprint`)

### Quá trình suy luận
- Kiến trúc RISC-V Sv39 sử dụng bảng trang 3 cấp (level 2 là root, level 1 là middle, level 0 là leaf), mỗi bảng trang có 512 PTE (`2^9`).
- Một PTE hợp lệ khi bit `PTE_V` được bật. Nếu một trong các bit quyền `PTE_R | PTE_W | PTE_X` được bật, đó là **leaf PTE** (trỏ tới trang vật lý hoặc superpage); ngược lại, nếu các bit quyền đều bằng 0 và `level > 0`, PTE đó trỏ tới một bảng trang cấp thấp hơn.
- Địa chỉ ảo tương ứng với mỗi PTE ở cấp `level` được tính bằng cách ghép chỉ số `i` dịch trái `PXSHIFT(level)` bit vào địa chỉ ảo tích lũy `va`: `child_va = va | ((uint64)i << PXSHIFT(level))`.
- Định dạng in yêu cầu:
  - Dòng đầu tiên: `page table <địa chỉ pagetable>`
  - Mỗi PTE hợp lệ in trên một dòng với thụt lề `" .."` lặp lại `2 - level + 1` lần, theo sau là `<va>: pte <pte> pa <pa>`, và các ký tự quyền `R`, `W`, `X`, `U` nếu là leaf PTE.
  - Nếu là superpage (leaf PTE ở `level > 0`), in thêm ` (superpage)` để phân biệt.

### Giải pháp
- Cài đặt hàm đệ quy `vmprint_walk(pagetable_t pagetable, int level, uint64 va)` và hàm `vmprint(pagetable_t pagetable)` trong `kernel/vm.c` sử dụng `printk` với định dạng `%p` cho con trỏ 64-bit.

### Kết quả kiểm thử
Chạy lệnh `./grade-lab-pgtbl vmprint`:
```
== Test pgtbltest == (1.5s) 
== Test   pgtbltest: vmprint_test == 
  pgtbltest: vmprint_test: OK 
```

---

## 3. Bài 3: Sử dụng superpages (`superpg`)

### Quá trình suy luận
- Trong RISC-V Sv39, ngoài trang chuẩn 4KB (`PGSIZE = 4096`), phần cứng hỗ trợ **megapage (superpage)** kích thước 2MB (`SUPERPGSIZE = 2 * 1024 * 1024`) bằng cách đặt một leaf PTE ngay tại **level 1** (thay vì trỏ tới bảng trang level 0).
- Điều kiện để ánh xạ một vùng nhớ bằng superpage 2MB:
  1. Địa chỉ ảo hiện tại `a` chia hết cho `SUPERPGSIZE` (`(a % SUPERPGSIZE) == 0`).
  2. Địa chỉ vật lý hiện tại `pa` chia hết cho `SUPERPGSIZE` (`(pa % SUPERPGSIZE) == 0`).
  3. Kích thước còn lại `size >= SUPERPGSIZE`.
- Để không ảnh hưởng đến bảng trang của user space (`mappages`), ta tách riêng một hàm `kmappages` dành cho `kvmmap` sử dụng.
- Cần hàm `walk_level(pagetable_t pagetable, uint64 va, int alloc, int target_level)` để duyệt từ level 2 xuống `target_level` (1 cho superpage, 0 cho trang 4KB).

### Giải pháp
- Thêm `walk_level` và `kmappages` vào `kernel/vm.c`.
- Trong `kmappages`, lặp qua vùng nhớ với `while (size > 0)`: nếu thỏa mãn cả 3 điều kiện căn chỉnh và kích thước 2MB thì gọi `walk_level(pagetable, a, 1, 1)` để gán leaf PTE ở level 1 và tăng `a`, `pa` thêm `SUPERPGSIZE`, giảm `size` đi `SUPERPGSIZE`; ngược lại ánh xạ từng trang 4KB ở level 0.
- Cập nhật `kvmmap` để gọi `kmappages`.
- Cập nhật `vmprint_walk` để hiển thị ` (superpage)` khi gặp leaf PTE ở `level > 0`.

### Kết quả kiểm thử
- Khi chạy `./grade-lab-pgtbl superpg` trên nhánh gốc, `pgtbltest` gọi `ugetpid_test()` trước `ksuper_test()` (mà `ugetpid` thuộc phần bài khác chưa cài đặt nên dừng ở `ugetpid_test`).
- Khi kiểm tra trực tiếp `ksuper_test()` (đặt trước `ugetpid_test()` để chạy thử rồi hoàn tác `user/pgtbltest.c` về nguyên trạng):
```
== Test pgtbltest == (1.4s) 
== Test   pgtbltest: superpg == 
  pgtbltest: superpg: OK 
```
