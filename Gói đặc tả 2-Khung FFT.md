# Gói đặc tả #2 — Khung FFT đối chứng

---

## 1. Mục đích

Gói #2 mô tả cách bộ giải mã FFT chia khung tín hiệu và trích công suất phổ,
để dùng làm **chuẩn đối chứng** khi so sánh độ chính xác với hai bộ giải mã
còn lại (Goertzel, ngân hàng bộ lọc). Cả ba bộ giải mã dùng chung một hàm
chia khung (`dtmf_segment`) và một luật quyết định (`dtmf_decide`), nên phải
thống nhất tham số ngay từ gói đặc tả này. Toàn bộ công thức bên dưới được
đối chiếu trực tiếp với giáo trình *Basic Digital Signal Processing* để đảm
bảo đúng ký hiệu và định nghĩa chuẩn.

## 2. Cơ sở lý thuyết (đối chiếu giáo trình)

### 2.1. Định nghĩa DFT

Với chuỗi hữu hạn `x(n)`, `0 ≤ n ≤ N−1`, hệ số DFT được định nghĩa (giáo
trình, Chương 4, Phương trình 8, tr. 113):

```
X(k) = Σ(n=0..N-1) x(n) * e^(-j*2*pi*k*n/N),     k = 0, 1, ..., N-1
```

trong đó `k` được giáo trình gọi là **chỉ số bin tần số** (*frequency bin
number*, Chương 4, mục 4.1.3).

### 2.2. Vì sao phải dùng cửa sổ (spectral leakage)

Giáo trình (Mục 4.6, tr. 129–131) chỉ ra: DFT ngầm giả định đoạn tín hiệu
được lấy là **tuần hoàn**. Nếu độ dài khung `N` không phải bội số nguyên của
chu kỳ tín hiệu, khi nối khung này với chính nó sẽ xuất hiện **điểm gián
đoạn biên độ**, sinh ra các thành phần tần số giả (không có trong tín hiệu
gốc) — gọi là **rò rỉ phổ (spectral leakage)**. Biên độ gián đoạn càng lớn,
rò rỉ càng nhiều.

Vì hai tần số DTMF trong một khung hiếm khi tạo đủ số chu kỳ nguyên trong
N = 256 mẫu, khung FFT đối chứng của dự án **bắt buộc phải nhân cửa sổ**
trước khi biến đổi Fourier, đúng theo khuyến nghị của giáo trình, để giảm
gián đoạn ở hai biên khung và giảm rò rỉ phổ.

### 2.3. Phép nhân cửa sổ

Giáo trình định nghĩa (Phương trình 32, tr. 131):

```
xw(n) = x(n) * w(n),     0 ≤ n ≤ N-1
```

tức nhân từng mẫu tín hiệu với hàm cửa sổ tương ứng trước khi đưa vào FFT.

### 2.4. Cửa sổ Hamming

Giáo trình liệt kê cửa sổ Hamming trong nhóm "các hàm cửa sổ thông dụng"
(Mục 4.6, Phương trình 35, tr. 132):

```
w_hm(n) = 0.54 - 0.46*cos(2*pi*n / (N-1)),     0 ≤ n ≤ N-1
```

Công thức này **khớp hoàn toàn** với công thức đã dùng trong
`dtmf_decode_fft.m`, không cần điều chỉnh gì.

### 2.5. Ánh xạ bin sang tần số và độ phân giải

Giáo trình định nghĩa (Phương trình 26/30, tr. 125):

```
f = k * fs / N     (Hz)
```

và độ phân giải tần số — khoảng cách giữa hai bin liên tiếp (Phương trình
31, tr. 126):

```
Δf = fs / N     (Hz)
```

Với N = 256, fs = 8000 Hz: **Δf = 8000/256 = 31,25 Hz**, đúng như đã dùng
trong bảng bin ở Mục 4 bên dưới.

### 2.6. Phổ công suất (power spectrum)

Giáo trình định nghĩa phổ công suất DFT một cách chuẩn tắc là (Phương trình
28, tr. 125):

```
P(k) = (1/N^2) * |X(k)|^2,     k = 0, 1, ..., N-1
```

tức có **chuẩn hóa theo 1/N²**. Trong `dtmf_decode_fft.m`, dự án dùng trực
tiếp `E(k) = |X(k)|^2` (không nhân 1/N²). Đây **không phải sai lệch với lý
thuyết**: vì hằng số 1/N² giống nhau cho mọi bin trong cùng một khung, nó
triệt tiêu khi `dtmf_decide` so sánh các bin với nhau theo **tỷ số** (đỉnh
so với đỉnh nhì, twist tính bằng 10·log10 của tỷ số công suất). Do đó dùng
`|X(k)|^2` thô cho kết quả quyết định giống hệt như dùng `P(k)` đã chuẩn
hóa, nhưng tiết kiệm một phép chia mỗi bin. Điểm này cần ghi rõ trong gói
đặc tả để không ai nhầm là code thiếu bước chuẩn hóa.

## 3. Tham số khung đã chốt

| Tham số | Giá trị | Ý nghĩa |
|---|---|---|
| `frameN` (N) | 256 mẫu | Số điểm FFT; là lũy thừa của 2 để tận dụng thuật toán chia-để-trị, cho tốc độ O(N log N) |
| Cửa sổ | Hamming, `w_hm(n)` (Ptr. 35, giáo trình) | Giảm rò rỉ phổ (spectral leakage) trước khi biến đổi Fourier |
| `hop` | 128 mẫu | Bước nhảy giữa hai khung liên tiếp, tức chồng lấp 50% |
| fs | 8000 Hz | Tần số lấy mẫu của toàn dự án |
| Δf = fs/N (Ptr. 31) | 8000/256 = **31,25 Hz** | Độ phân giải tần số của phổ FFT |

## 4. Bảng bin k và % sai lệch (đủ 7 tần số chuẩn)

Bin `k` là chỉ số nguyên gần nhất ứng với tần số chuẩn `f`, suy ra từ công
thức ánh xạ bin–tần số của giáo trình (Ptr. 26): `f = k*fs/N` ⇒
`k = round(N*f/fs)`.

```
k = round(N * f / fs)
f_k_thuc_te = k * fs / N          (áp dụng đúng Phương trình 26)
sai_lech(%) = (f_k_thuc_te - f) / f * 100
```

| Nhóm | Tần số chuẩn (Hz) | k = round(N·f/fs) | f_k thực tế (Hz) | Sai lệch (%) |
|---|---:|---:|---:|---:|
| Hàng 1 | 697  | 22 | 687,500  | −1,36% |
| Hàng 2 | 770  | 25 | 781,250  | +1,46% |
| Hàng 3 | 852  | 27 | 843,750  | −0,97% |
| Hàng 4 | 941  | 30 | 937,500  | −0,37% |
| Cột 1  | 1209 | 39 | 1218,750 | +0,81% |
| Cột 2  | 1336 | 43 | 1343,750 | +0,58% |
| Cột 3  | 1477 | 47 | 1468,750 | −0,56% |

**Nhận xét:** sai lệch lớn nhất là **+1,46%** (tần số 770 Hz), vẫn nằm trong
dung sai **nhận ≤ ±1,5%** của ITU-T Q.24 → khung 256 điểm hợp lệ để dùng làm
đối chứng.

## 5. Bin hài bậc 2

Ngoài 7 bin chuẩn trên, `info.E` cần thêm 1 bin thứ 8 là hài bậc 2 của tần
số mạnh nhất trong khung (bin `2k` của tần số đang trội, cũng suy từ
`f = k*fs/N` ở Ptr. 26), dùng để loại khung nghi là tiếng nói
(`reject = 'harmonic'`). Bin này **không cố định** vì phụ thuộc tần số trội
của từng khung, nên không đưa vào bảng tĩnh ở Mục 4.

## 6. Việc cần làm khi cài đặt (`dtmf_decode_fft.m`)

1. `seg = dtmf_segment(y, 'fs',fs, 'frameN',256, 'hop',128);`
2. Với mỗi khung: nhân cửa sổ Hamming theo Mục 2.3–2.4:
   `xw = frame .* w_hm`, rồi `X = fft(xw, 256)`.
3. Lấy `E(j,i) = |X(k_j)|^2` tại 8 bin (7 bin ở Mục 4 + 1 bin hài ở Mục 5) —
   xem giải thích ở Mục 2.6 vì sao không cần nhân thêm 1/N².
4. `[rowIdx, colIdx, conf, reject] = dtmf_decide(E(:,i));`
5. Chống dội (debounce): gộp các khung liên tiếp cùng phím thành một ký tự.

## 7. Tham khảo

- [1] *Basic Digital Signal Processing*, Chương 4 "Discrete Fourier
  Transform and Signal Spectrum", tr. 104–133 — dùng cho định nghĩa DFT
  (Ptr. 8), ánh xạ bin–tần số (Ptr. 26/30), độ phân giải tần số (Ptr. 31),
  phổ công suất (Ptr. 28), và cửa sổ Hamming (Mục 4.6, Ptr. 35).
- [2] F. J. Harris, "On the use of windows for harmonic analysis with the
  discrete Fourier transform," *Proc. IEEE*, vol. 66, no. 1, pp. 51–83, 1978.
- [3] `CONTRACTS.md` — Thông số chốt sẵn của dự án.
- [4] `src/decode/dtmf_decode_fft.m` — chữ ký hàm và TODO cài đặt.

---
*Deliverable Tuần 1 (hạn 18/09): bảng bin k cho đủ 7 tần số kèm % sai lệch —
xem Mục 4. Đã hoàn thành và đối chiếu đúng ký hiệu/công thức của giáo trình
Basic Digital Signal Processing.*
