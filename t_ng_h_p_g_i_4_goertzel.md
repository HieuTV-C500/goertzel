# GÓI #4: THUẬT TOÁN GOERTZEL - CÔNG THỨC, BẢNG THAM SỐ VÀ VÍ DỤ TÍNH TAY

## 1. Công thức truy hồi Goertzel
Thuật toán Goertzel được sử dụng để tính toán năng lượng (hoặc biên độ) tại một tần số cụ thể (bin $k$) thay vì tính toán toàn bộ phổ như FFT.

**Các bước thực hiện:**
Với tín hiệu đầu vào $x[n]$ có độ dài $N$ mẫu, tần số lấy mẫu $f_s$, và tần số cần phát hiện $f$:

*   **Bước 1: Tính chỉ số bin $k$**
    $$k = \text{round}\left( \frac{f \cdot N}{f_s} \right)$$
*   **Bước 2: Tính hệ số góc và hệ số bộ lọc (Coefficient)**
    $$\omega_k = \frac{2\pi k}{N}$$
    $$C = 2 \cos(\omega_k)$$
*   **Bước 3: Vòng lặp truy hồi (Feedback Phase)**
    Khởi tạo các trạng thái ban đầu: $s[-1] = 0, s[-2] = 0$.
    Với mỗi mẫu $n$ từ $0$ đến $N-1$, cập nhật trạng thái:
    $$s[n] = x[n] + C \cdot s[n-1] - s[n-2]$$
*   **Bước 4: Tính công suất (Feedforward Phase)**
    Sau khi chạy hết $N$ mẫu, công suất $P$ tại tần số mục tiêu được tính bằng hai trạng thái cuối cùng ($s[N-1]$ và $s[N-2]$):
    $$P = s[N-1]^2 + s[N-2]^2 - C \cdot s[N-1] \cdot s[N-2]$$

---

## 2. Bảng tham số $N = 205$ & $k$
Trong viễn thông, hệ thống DTMF sử dụng tần số lấy mẫu phổ biến $f_s = 8000$ Hz. Với $N = 205$, ta tính được các giá trị $k$ tương ứng cho 8 tần số cơ bản của DTMF như sau:

| Loại | Tần số $f$ (Hz) | Tính toán $k_{exact} = \frac{f \cdot 205}{8000}$ | Giá trị $k$ (làm tròn) |
| :--- | :--- | :--- | :--- |
| **Row 1** | 697 | 17.8606 | **18** |
| **Row 2** | 770 | 19.7312 | **20** |
| **Row 3** | 852 | 21.8325 | **22** |
| **Row 4** | 941 | 24.1131 | **24** |
| **Col 1** | 1209 | 30.9806 | **31** |
| **Col 2** | 1336 | 34.2350 | **34** |
| **Col 3** | 1477 | 37.8481 | **38** |
| **Col 4** | 1633 | 41.8456 | **42** |

---

## 3. Hướng dẫn thiết lập file Excel tính tay ($N=16, k=3$)
Để chứng minh code ra khớp kết quả $P = 64$ với $N=16$ và $k=3$, bạn thiết lập bảng Excel với các cột như sau:

**Thông số ban đầu:**
*   $N = 16$
*   $k = 3$
*   $\omega_k = \frac{2 \pi \cdot 3}{16} \approx 1.178097$
*   $C = 2 \cos(\omega_k) = 2 \cos(1.178097) \approx 0.765367$

**Cấu trúc các cột trong Excel:**
*   **Cột A (n):** Từ 0 đến 15.
*   **Cột B (x[n]):** Giả sử tín hiệu đầu vào là một hàm cosine đơn giản có biên độ bằng 1: $x[n] = \cos(\omega_k \cdot n)$.
*   **Cột C (s[n-1]):** Trạng thái trễ 1 bước (lấy từ cột E của hàng trên).
*   **Cột D (s[n-2]):** Trạng thái trễ 2 bước (lấy từ cột C của hàng trên).
*   **Cột E (s[n]):** Nhập công thức: `= B2 + $C$1 * C2 - D2` (Trong đó `$C$1` là ô chứa hằng số $C \approx 0.765367$).

Sau khi kéo công thức đến hàng $n=15$, bạn lấy 2 giá trị cuối cùng là $s[15]$ và $s[14]$ để tính $P$ theo công thức ở Bước 4. Kết quả sẽ xấp xỉ $64.000000$.