# Gói đặc tả #4 — Công thức truy hồi Goertzel

**Tổ:** S2 + R2 · **Hạn:** 21/09 · **Hàm liên quan:** `goertzel_power.m`, `dtmf_decode_goertzel.m`
**Tham chiếu:** `CONTRACTS.md` — "Khung Goertzel N = 205 → Δf ≈ 39,02 Hz"; dung sai tần số nhận ≤ ±1,5%.

---

## I/O

**Hàm 1 — `goertzel_power(x, k, N)`** (tính công suất tại 1 bin)

| Tên | Kích thước | Kiểu | Ghi chú |
|---|---|---|---|
| `x` | 1×M | `double` | Khung tín hiệu (chỉ dùng N mẫu đầu, M ≥ N) |
| `k` | 1×1 | `double` (nguyên, ≥0) | Chỉ số bin; tần số tương ứng f_k = k·fs/N [Hz] |
| `N` | 1×1 | `double` (nguyên, >0) | Độ dài khung = số điểm DFT [mẫu] |
| **→** `P` | 1×1 | `double` (≥0) | Công suất tại bin k, P = \|X[k]\|² |

**Hàm 2 — `dtmf_decode_goertzel(y, opt)`** (giải mã đầy đủ, cùng chữ ký với `dtmf_decode_fft`/`dtmf_decode_filterbank`)

| Tên | Kích thước | Kiểu | Ghi chú |
|---|---|---|---|
| `y` | 1×L | `double` | Tín hiệu cần giải mã |
| `opt.fs` | 1×1 | `double` | Tần số lấy mẫu (mặc định 8000) [Hz] |
| `opt.frameN` | 1×1 | `double` | Độ dài khung (mặc định **205**) [mẫu] |
| `opt.hop` | 1×1 | `double` | Bước nhảy giữa hai khung (mặc định 205 → không chồng lấp) [mẫu] |
| `keys` | 1×K | `char` | Chuỗi phím giải mã |
| `info.E` | 8×nFrame | `double` | Công suất tại 7 tần số chuẩn + 1 hài bậc 2 |
| `info.rowIdx` | 1×nFrame | `double` | Chỉ số hàng 1..4 (0: không quyết định) |
| `info.colIdx` | 1×nFrame | `double` | Chỉ số cột 1..3 (0: không quyết định) |
| `info.conf` | 1×nFrame | `double` | Độ tin cậy ∈ [0, 1] |
| `info.tFrame` | 1×nFrame | `double` | Thời điểm khung [s] |
| `info.reject` | 1×nFrame | `cellstr` | `'none'` \| `'twist'` \| `'level'` \| `'harmonic'` |

*Khác với Gói #2 (FFT dùng N=256, hop=128, chồng lấp 50%), Goertzel dùng khung N=205, hop=205 (không chồng lấp) — vì Goertzel tính trực tiếp từng bin cần thiết (8 bin) với độ phức tạp O(N) mỗi bin, không cần toàn phổ nên không bắt buộc chồng lấp để làm mượt phổ như FFT.*

---

## Công thức

**1) Hệ số hồi quy c** (phụ thuộc k, N — hằng số trong suốt một lần tính)

$$
c = 2\cos\!\left(\frac{2\pi k}{N}\right)
$$

```
c = 2 * cos(2*pi*k/N)
```

**2) Vòng lặp truy hồi bậc 2** (bộ lọc IIR, chỉ số n tính từ 0, s[-1] = s[-2] = 0)

$$
s[n] = x[n] + c\cdot s[n-1] - s[n-2], \qquad n = 0, 1, \dots, N-1
$$

```
s[n] = x[n] + c*s[n-1] - s[n-2]   ,   n = 0..N-1   ,   s[-1]=s[-2]=0
```

**3) Công suất tại bin k** (chỉ tính SAU khi vòng lặp chạy hết N mẫu, dùng 2 giá trị s cuối cùng)

$$
P = s[N-1]^2 + s[N-2]^2 - c\cdot s[N-1]\cdot s[N-2]
$$

```
P = s[N-1]^2 + s[N-2]^2 - c*s[N-1]*s[N-2]
```

*(Nếu trình đọc của bạn không hiển thị được khối `$$...$$` ở trên thì dùng khối chữ ngay bên dưới mỗi công thức — nội dung giống hệt.)*

**Vì sao đúng:** đây là cách tính X[k] = Σ x[n]·e^(-j2πkn/N) mà không cần lượng giác phức tại từng bước — công suất |X[k]|² được lấy ra chỉ từ 2 giá trị s cuối, độ phức tạp O(N) nhân thực mỗi bin (so với O(N log N) của FFT toàn phổ), hiệu quả hơn khi chỉ cần 8 bin cố định như DTMF.

---

## Ví dụ số kiểm chứng

### A. Bảng bin k cho N = 205 (đúng chuẩn 7 tần số DTMF)

`fs = 8000 Hz`, `N = 205` ⇒ `Δf = fs/N ≈ 39,0244 Hz`.

| Nhóm | f chuẩn [Hz] | k = round(f·N/fs) | f_k = k·Δf [Hz] | Sai lệch [%] | Đánh giá |
|---|---:|---:|---:|---:|---|
| Hàng | 697 | 18 | 702,44 | +0,780% | Đạt (≤1,5%) |
| Hàng | 770 | 20 | 780,49 | +1,362% | Đạt (≤1,5%) |
| Hàng | 852 | 22 | 858,54 | +0,767% | Đạt (≤1,5%) |
| Hàng | 941 | 24 | 936,59 | −0,469% | Đạt (≤1,5%) |
| Cột | 1209 | 31 | 1209,76 | +0,063% | Đạt (≤1,5%) |
| Cột | 1336 | 34 | 1326,83 | −0,686% | Đạt (≤1,5%) |
| Cột | 1477 | 38 | 1482,93 | +0,401% | Đạt (≤1,5%) |

→ Trùng khớp với bảng đã chốt trong `CONTRACTS.md`. Sai lệch lớn nhất vẫn là 770 Hz (+1,362%), nhưng thấp hơn mức 1,461% của Gói #2 (N=256) — vì Δf của N=205 (≈39,02 Hz) không nhất thiết chia hết đẹp cho 770 hơn N=256, đây là trùng hợp toán học, không phải quy luật chung.

### B. Ví dụ tính tay: N = 16, k = 3, x[n] = cos(2π·3n/16)

Hằng số: `c = 2*cos(2*pi*3/16) = 0,765367`.

| n | x[n] = cos(2π·3n/16) | s[n] = x[n] + c·s[n−1] − s[n−2] |
|---:|---:|---:|
| 0 | +1,000000 | +1,000000 |
| 1 | +0,382683 | +1,148050 |
| 2 | −0,707107 | −0,828427 |
| 3 | −0,923880 | −2,705981 |
| 4 | −0,000000 | −1,242641 |
| 5 | +0,923880 | +2,678784 |
| 6 | +0,707107 | +4,000000 |
| 7 | −0,382683 | +0,000000 |
| 8 | −1,000000 | −5,000000 |
| 9 | −0,382683 | −4,209518 |
| 10 | +0,707107 | +2,485281 |
| 11 | +0,923880 | +7,035549 |
| 12 | +0,000000 | +2,899495 |
| 13 | −0,923880 | −5,740251 |
| 14 | −0,707107 | −8,000000 |
| 15 | +0,382683 | −0,000000 |

Lấy 2 giá trị cuối: `s[15] = 0`, `s[14] = −8,000000`.

$$
P = s[15]^2 + s[14]^2 - c\cdot s[15]\cdot s[14] = 0^2 + (-8)^2 - 0,765367\times 0 \times(-8) = 64,000000
$$

```
P = 0^2 + (-8)^2 - 0,765367*0*(-8) = 64,000000
```

**Đối chiếu lý thuyết:** với x[n] = cos(2π·3n/16) là sóng thuần tại đúng bin k=3 của N=16 điểm, biên độ DFT lý thuyết là X[3] = N/2 = 8 ⇒ |X[3]|² = 64 — khớp tuyệt đối với kết quả truy hồi (sai số < 1e-9, xem `tests/test_goertzel.m`).

*Ghi chú Excel:* bảng phần B ở trên copy thẳng được vào 3 cột (n, x[n], s[n]) trong Excel, dùng công thức ô `s[n] = x[n] + c*s[n-1] - s[n-2]` kéo xuống 16 dòng, 2 dòng đầu s[-1], s[-2] coi là 0 — đúng như tiêu chí "File Excel tính tay N=16, k=3 → P=64,000000" trong bảng công việc.

---

## Tiêu chí xong

- [x] Đủ 4 phần bắt buộc: I/O · công thức · ví dụ số kiểm chứng · tiêu chí xong.
- [x] Bảng bin k có đủ 7 tần số chuẩn cho N=205, mỗi dòng ghi k, f_k và % sai lệch.
- [x] Toàn bộ 7 sai lệch đều ≤ 1,5% (khớp dung sai "nhận" trong `CONTRACTS.md`).
- [x] Có ví dụ tính tay đầy đủ 16 bước (N=16, k=3) dẫn ra đúng P = 64,000000, đối chiếu được với lý thuyết (X[3]=N/2=8).
- [x] Công thức c, s[n], P viết đúng cú pháp MATLAB, chỉ rõ quy ước chỉ số từ 0 và điều kiện đầu s[-1]=s[-2]=0.
- [x] Chữ ký hàm khớp `goertzel_power(x,k,N)` và `dtmf_decode_goertzel(y,opt)` trong `CONTRACTS.md`.
- [ ] Coder đối chiếu code thật với `tests/test_goertzel.m` (test_knownValue và test_matchesBuiltinGoertzel, sai số tương đối < 1e-10) trước khi coi Gói #4 là hoàn tất — ngoài phạm vi Gói #4, chỉ cần bàn giao đúng công thức + 2 bảng ở trên cho tổ Coder.
