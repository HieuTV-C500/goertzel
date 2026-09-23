# Gói đặc tả #4 — Công thức truy hồi Goertzel

---

## I/O

**Hàm 1 — `goertzel_power(x, k, N)`** (tính công suất tại 1 bin — hàm lõi, Coder cài đặt trực tiếp theo công thức bên dưới)

| Tên | Kích thước | Kiểu / ràng buộc | Ghi chú |
|---|---|---|---|
| `x` | 1×M | `double` | Khung tín hiệu (chỉ dùng N mẫu đầu, M $\ge$ N) |
| `k` | 1×1 | `double`, nguyên, $\ge 0$ (`mustBeInteger`, `mustBeNonnegative`) | Chỉ số bin; tần số tương ứng $f_k = k \cdot f_s / N$ [Hz] |
| `N` | 1×1 | `double`, nguyên, $> 0$ (`mustBeInteger`, `mustBePositive`) | Độ dài khung = số điểm DFT [mẫu] |
| **$\rightarrow$** `P` | 1×1 | `double`, $\ge 0$ | Công suất tại bin k, $P = \|X[k]\|^2$ |

Gọi hàm: `P = goertzel_power(x, k, N)` — dùng khối `arguments...end` để MATLAB tự kiểm tra kiểu/ràng buộc trước khi chạy thân hàm.

**Hàm 2 — `dtmf_decode_goertzel(y, opt)`** (giải mã đầy đủ, cùng chữ ký với `dtmf_decode_fft`/`dtmf_decode_filterbank` — xem `CONTRACTS.md`)

| Tên | Kích thước | Kiểu | Ghi chú |
|---|---|---|---|
| `y` | 1×N | `double` | Tín hiệu cần giải mã |
| `'fs'` | 1×1 | `double` | Tham số tên–giá trị, tần số lấy mẫu (mặc định 8000) [Hz] |
| `'frameN'` | 1×1 | `double` | Tham số tên–giá trị, độ dài khung (mặc định **205**) [mẫu] |
| `'hop'` | 1×1 | `double` | Tham số tên–giá trị, bước nhảy giữa hai khung (mặc định 205 $\rightarrow$ không chồng lấp) [mẫu] |
| `keys` | 1×K | `char` | Chuỗi phím giải mã, theo thứ tự thời gian |
| `info.E` | 8×nFrame | `double` | Công suất tại 7 tần số chuẩn + 1 hài bậc 2, mỗi khung |
| `info.rowIdx` | 1×nFrame | `double` | Chỉ số hàng 1..4 (0: không quyết định) |
| `info.colIdx` | 1×nFrame | `double` | Chỉ số cột 1..3 (0: không quyết định) |
| `info.conf` | 1×nFrame | `double` | Độ tin cậy $\in [0, 1]$ |
| `info.tFrame` | 1×nFrame | `double` | Thời điểm của khung [s] |
| `info.reject` | 1×nFrame | `cellstr` | `'none'` \| `'twist'` \| `'level'` \| `'harmonic'` |

**Cách gọi thực tế** (khối `arguments` khai báo `opt.fs`, `opt.frameN`, `opt.hop` nên đây là **tham số tên–giá trị**, không phải một struct `opt` truyền trực tiếp):

```matlab
[keys, info] = dtmf_decode_goertzel(y);                                   % dùng mặc định
[keys, info] = dtmf_decode_goertzel(y, 'fs',8000, 'frameN',205, 'hop',205); % tùy chỉnh
```

**Bin hài bậc 2** (dòng thứ 8 của `info.E`, TODO của Coder theo `dtmf_decode_goertzel.m`): lấy tại bin `2k` của tần số mạnh nhất trong khung (hoặc một ước lượng đơn giản tương đương) — dùng cho tiêu chí loại `'harmonic'` trong `info.reject`, không thuộc phạm vi tính toán chính của Gói #4, chỉ nêu để Coder biết vị trí cần điền.

*So với Gói #2 (FFT dùng N=256, hop=128, chồng lấp 50%): Goertzel dùng khung N=205, hop=205 (không chồng lấp) vì chỉ cần tính trực tiếp 8 bin cố định (độ phức tạp $\mathcal{O}(N)$ mỗi bin), không cần dựng toàn phổ nên không bắt buộc chồng lấp để làm mượt phổ như FFT.*

---

## Công thức

**1) Hệ số hồi quy $c$** (phụ thuộc $k$, $N$ — hằng số trong suốt một lần tính)

$$
c = 2\cos\!\left(\frac{2\pi k}{N}\right)
$$

```matlab
c = 2 * cos(2*pi*k/N)
```

**2) Vòng lặp truy hồi bậc 2** (bộ lọc IIR, chỉ số $n$ tính từ 0, $s[-1] = s[-2] = 0$)

$$
s[n] = x[n] + c\cdot s[n-1] - s[n-2], \qquad n = 0, 1, \dots, N-1
$$

```matlab
s[n] = x[n] + c*s[n-1] - s[n-2]   % n = 0..N-1, s[-1]=s[-2]=0
```

**3) Công suất tại bin $k$** (chỉ tính SAU KHI vòng lặp chạy hết $N$ mẫu, dùng 2 giá trị $s$ cuối cùng)

$$
P = s[N-1]^2 + s[N-2]^2 - c\cdot s[N-1]\cdot s[N-2]
$$

```matlab
P = s[N-1]^2 + s[N-2]^2 - c*s[N-1]*s[N-2]
```

**Vì sao đúng:** đây là cách tính $X[k] = \sum x[n]\cdot e^{-j2\pi kn/N}$ mà không cần lượng giác phức tại từng bước — công suất $\|X[k]\|^2$ được lấy ra chỉ từ 2 giá trị $s$ cuối. Độ phức tạp $\mathcal{O}(N)$ phép nhân thực mỗi bin, so với $\mathcal{O}(N \log N)$ của FFT toàn phổ — hiệu quả hơn khi chỉ cần 8 bin cố định như DTMF, đổi lại không cho ra toàn bộ phổ như FFT (không dùng để vẽ spectrogram được).

**Quy ước chỉ số (theo `CONTRACTS.md` §"Quy ước chú thích"):** $n$ trong công thức toán tính từ 0; khi cài đặt trong MATLAB, $x[n]$ ứng với `x(n+1)` (mảng MATLAB 1-based).

---

## Ví dụ số kiểm chứng

### A. Bảng bin k cho N = 205 (đúng chuẩn 7 tần số DTMF)

$f_s = 8000 \text{ Hz}$, $N = 205 \Rightarrow \Delta f = f_s/N \approx 39,0244 \text{ Hz}$.

| Nhóm | $f$ chuẩn [Hz] | $k = \text{round}(f \cdot N/f_s)$ | $f_k = k \cdot \Delta f$ [Hz] | Sai lệch [%] | Đánh giá |
|---|---:|---:|---:|---:|---|
| Hàng | 697 | 18 | 702,44 | +0,780% | Đạt ($\le 1,5\%$) |
| Hàng | 770 | 20 | 780,49 | +1,362% | Đạt ($\le 1,5\%$) |
| Hàng | 852 | 22 | 858,54 | +0,767% | Đạt ($\le 1,5\%$) |
| Hàng | 941 | 24 | 936,59 | −0,469% | Đạt ($\le 1,5\%$) |
| Cột | 1209 | 31 | 1209,76 | +0,063% | Đạt ($\le 1,5\%$) |
| Cột | 1336 | 34 | 1326,83 | −0,686% | Đạt ($\le 1,5\%$) |
| Cột | 1477 | 38 | 1482,93 | +0,401% | Đạt ($\le 1,5\%$) |

$\rightarrow$ Trùng khớp với bảng đã chốt trong `CONTRACTS.md` §"Thông số chốt sẵn". Sai lệch lớn nhất vẫn là 770 Hz (+1,362%), thấp hơn mức 1,461% của Gói #2 (N=256).

### B. Ví dụ tính tay: N = 16, k = 3, x[n] = cos(2π·3n/16)

Hằng số: $c = 2\cos\left(\frac{2\pi \cdot 3}{16}\right) = 0,765367$.

| n | $x[n] = \cos(2\pi\cdot 3n/16)$ | $s[n] = x[n] + c\cdot s[n-1] - s[n-2]$ |
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

Lấy 2 giá trị cuối: $s[15] = 0$, $s[14] = -8,000000$.

$$
P = s[15]^2 + s[14]^2 - c\cdot s[15]\cdot s[14] = 0^2 + (-8)^2 - 0,765367 \times 0 \times (-8) = 64,000000
$$

**Đối chiếu lý thuyết:** với $x[n] = \cos(2\pi\cdot 3n/16)$ là sóng thuần tại đúng bin $k=3$ của $N=16$ điểm, biên độ DFT lý thuyết là $X[3] = N/2 = 8 \Rightarrow \|X[3]\|^2 = 64$ — khớp tuyệt đối với kết quả truy hồi.

---

## Tiêu chí xong

- [x] Đủ 4 phần bắt buộc: I/O · công thức · ví dụ số kiểm chứng · tiêu chí xong.
- [x] Bảng bin k có đủ 7 tần số chuẩn cho N=205, mỗi dòng ghi k, f_k và % sai lệch.
- [x] Toàn bộ 7 sai lệch đều $\le 1,5\%$ (khớp dung sai "nhận" trong `CONTRACTS.md`); không có dòng nào cần cảnh báo hoặc bị từ chối.
- [x] Có ví dụ tính tay đầy đủ 16 bước (N=16, k=3) dẫn ra đúng P = 64,000000, đối chiếu được với lý thuyết (X[3]=N/2=8), kèm file Excel công thức sống.
- [x] Công thức c, s[n], P viết đúng cú pháp MATLAB, chỉ rõ quy ước chỉ số từ 0 và điều kiện đầu s[-1]=s[-2]=0, cùng đơn vị `[Hz]`, `[mẫu]` cho mọi biến I/O.
- [x] Chữ ký hàm và ràng buộc kiểu khớp đúng khối `arguments...end` trong `goertzel_power.m`/`dtmf_decode_goertzel.m`, cùng dạng tham số tên–giá trị với `dtmf_decode_fft`/`dtmf_decode_filterbank` (`CONTRACTS.md`).
- [x] Có mục Tham khảo dạng IEEE + tài liệu nội bộ theo đúng quy ước comment của nhóm.
- [ ] Coder cài đặt xong `goertzel_power.m` và đối chiếu với `tests/test_goertzel.m`:
  - `test_knownValue`: N=16, k=3 $\rightarrow$ P = 64.0 (`AbsTol` 1e-9).
  - `test_matchesBuiltinGoertzel`: N=205, k=18 (~697 Hz), tín hiệu ngẫu nhiên (`rng(1)`) $\rightarrow$ so với hàm `goertzel()` có sẵn của MATLAB (`RelTol` 1e-10).

---

## Tham khảo

- [1] G. Goertzel, "An algorithm for the evaluation of finite trigonometric series," *Amer. Math. Monthly*, vol. 65, no. 1, pp. 34–35, 1958.
- [2] J. G. Proakis, D. G. Manolakis, *Digital Signal Processing: Principles, Algorithms, and Applications*, 4th ed., Pearson, 2007.
- [3] Gói đặc tả #4 (tổ S2 + R2) — tài liệu này.
